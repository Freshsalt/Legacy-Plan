# 控制代码

> <font size=5>代码总览</font>

---

<font size=5>&emsp;&emsp;我将自己写了5个月的代码(4.2.py)托付在这里，希望后者能够在此基础上继续努力。\
</font>

    from ctypes import *
    import numpy as np
    import cv2
    import paddle
    import paddle.fluid as fluid
    from PIL import Image
    import sys
    import os
    import torch
    import serial
    from models.experimental import attempt_load
    from utils.datasets import letterbox
    from utils.general import non_max_suppression
    from utils.torch_utils import select_device
    from multiprocessing import Process, Queue
    import time
    from datetime import datetime

    paddle.enable_static()

    flag = 0
    flag_red = 0
    flag_crossing = 0
    flag_right = 0
    flag_left = 0
    flag_green = 0
    flag_attention = 0


    lib_path = "../lib/libart_driver.so"
    so = cdll.LoadLibrary
    lib = so(lib_path)
    car_port = "/dev/0"

    if lib.art_racecar_init(9600, car_port.encode("utf-8")) < 0:
        print("初始化赛车时出错。")


    def dataset(frame):
        lower_yellow = np.array([23, 43, 46])
        upper_yellow = np.array([37, 255, 255])
        lower_red = np.array([0, 40, 40]) 
        upper_red = np.array([10, 255, 255])
        hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
        mask_yellow = cv2.inRange(hsv, lower_yellow, upper_yellow)
        mask_red = cv2.inRange(hsv, lower_red, upper_red)
        mask = cv2.bitwise_or(mask_yellow, mask_red)
        img = Image.fromarray(mask)
        img = img.resize((120, 120), Image.ANTIALIAS)
        img = np.array(img).astype(np.float32)
        img = cv2.cvtColor(img, cv2.COLOR_GRAY2BGR)
        img = img.transpose((2, 0, 1))
        img = img[(2, 1, 0), :, :] / 255.0
        img = np.expand_dims(img, axis=0)
        return img


    def car_control(velocity, steering_angle):
        lib.send_cmd(velocity, steering_angle)


    default_velocity = 1545
    default_steering = 1500
    t = 0
    command_num = 0
    command_queue = Queue(2)  
    serial_connection = None


    def follow_lane():
        global (default_velocity, default_steering, command_num, t, command_queue, command_lock,
            flag_right, flag_left, )

        place = fluid.CPUPlace()
        exe = fluid.Executor(place)
        infer_program, feeded_var_names, target_vars = fluid.io.load_inference_model(
            dirname="../model/model_infer/", executor=exe)

        cap = cv2.VideoCapture(1)
        if not cap.isOpened():
            print('摄像头1不可用。')
            return

        while True:
            ret, frame = cap.read()
            if not ret:
                print('读取帧失败。')
                continue

            img = dataset(frame)
            result = exe.run(program=infer_program, feed={feeded_var_names[0]: img}, fetch_list=target_vars)
            model_steering = int(result[0][0][0] + 0.5)

            if flag_left == 1 and flag_right == 1:
                if model_steering > 1760:
                    model_steering = int(model_steering * 1.00)
                if model_steering < 1400:
                    model_steering = int(model_steering * 0.96)
            elif flag_left == 0 and flag_right == 0:
                if model_steering > 1500:
                    model_steering = int(model_steering * 1.8)
                if model_steering < 1400:
                    model_steering = int(model_steering * 0.97)
            elif flag_park == 1:
                if model_steering > 1700:
                    model_steering = int(model_steering * 1.7)
                if 1500 < model_steering <= 1700:
                    model_steering = int(model_steering * 1.01)
                if model_steering < 1500:
                    model_steering = int(model_steering * 0.8)
            else:
                if model_steering > 1650:
                    model_steering = int(model_steering * 1.5)

            model_steering = max(1100, min(model_steering, 1900))

            if not command_queue.empty():
                cmd_vel, cmd_angle = command_queue.get()
                handle_commands(cmd_vel, cmd_angle)

            if flag_roundabout:
                elapsed_time = (datetime.now() - roundabout_start_time).total_seconds()
                if elapsed_time >= 10:
                    if cmd_angle == 1900:
                        car_control(default_velocity, 1900)
                        time.sleep(1.2)
                        print("exit from left roundabout")
                    elif cmd_angle == 1100:
                        car_control(default_velocity, 1100)
                        time.sleep(0.9)
                        print("exit from right roundabout")
                    flag_roundabout = False
                    roundabout_start_time = None

            if flag_roundabout_2:
                elapsed_time_2 = (datetime.now() - roundabout_start_time_2).total_seconds()
                if elapsed_time_2 >= 50:
                    default_velocity = 1500
                    end_flag = True
                    print("The END!!!")
                    flag_roundabout_2 = False
                    roundabout_start_time_2 = None

            if command_num == 0:
                car_control(default_velocity, model_steering)
                if end_flag:
                    car_control(1500, 1500)


    def recognize_signs():
        global (default_velocity, default_steering, command_queue, command_lock, flag_attention,
            cancel_flag, right_buf, left_buf)

        device = select_device('gpu')
        half = device.type != 'gpu'
        weights = '../model/yolov5_model/best.pt'
        model = attempt_load(weights, map_location=device)
        names = model.module.names if hasattr(model, 'module') else model.names

        cap = cv2.VideoCapture(0)
        cap.set(cv2.CAP_PROP_FPS, 3)

        while True:
            ret, frame = cap.read()
            if not ret:
                print('读取帧失败。')
                continue

            with torch.no_grad():
                frame = cv2.addWeighted(frame, 1, frame, 0, 1)
                img = letterbox(frame, new_shape=320)[0]
                img = img[:, :, ::-1].transpose(2, 0, 1)
                img = np.ascontiguousarray(img)
                img = torch.from_numpy(img).to(device)
                img = img.half() if half else img.float()
                img /= 255.0
                if img.ndimension() == 3:
                    img = img.unsqueeze(0)

                pred = model(img, augment=False)[0]
            pred = non_max_suppression(pred, 0.4, 0.5, classes=False, agnostic=False)

            for i, det in enumerate(pred):
                if det is not None and len(det):
                    for *xyxy, conf, cls in reversed(det):
                        label = names[int(cls)]
                        conf = round(float(conf), 2)
                        width = int(xyxy[2]) - int(xyxy[0])
                        height = int(xyxy[3]) - int(xyxy[1])
                        area = width * height

                        if conf >= 0.50:
                            if label == 'cancel_10' and cancel_10_buf == 0:
                                if area >= 800:
                                    default_velocity = 1550
                                    command_queue.put((1551, 1500))
                                    cancel_10_buf = 1
                                    cancel_flag = 1
                            elif label == 'crossing' and crossing_buf == 0:
                                if area >= 1680:
                                    command_queue.put((1505, default_steering))
                                    crossing_buf = 1
                            elif label == 'limit_10' and limit_10_buf == 0:
                                command_queue.put((1540, default_steering))
                                limit_10_buf = 1
                            elif label == 'turn_left' and left_buf == 0:
                                if area >= 1320:
                                    if flag_attention == 1:
                                        command_queue.put((1509, default_steering))
                                    else:
                                        left_buf = 1
                                        command_queue.put((default_velocity, 1900))
                            elif label == 'attention' and attention_buf == 0:
                                if area >= 1270:
                                    command_queue.put((default_velocity, 1899))
                                    flag_attention = 1
                                    attention_buf = 1
                            elif label == 'paper_red' and cancel_flag == 1 and paper_red_buf == 0:
                                if area >= 600:
                                    command_queue.put((1506, default_steering))
                                    paper_red_buf = 1
                            elif label == 'paper_greend' and cancel_flag == 1 and paper_greend_buf == 0:
                                if area >= 800:
                                    command_queue.put((1552, default_steering))
                                    paper_greend_buf = 1
                            elif label == 'turn_right' and right_buf == 0:
                                if area >= 1320:
                                    if flag_attention == 1 and cancel_flag == 0:
                                        command_queue.put((1509, default_steering))
                                    else:
                                        right_buf = 1
                                        command_queue.put((default_velocity, 1100))
                            elif label == 'park' and cancel_flag == 1:
                                if area >= 10:
                                    command_queue.put((1553, default_steering))


    def handle_commands(command_velocity, command_steering):
        global (default_velocity, default_steering, command_num, t, flag, flag_red, flag_crossing,
            flag_green, flag_right)

        if command_velocity == 1505:
            command_num = 1
            print("crossing")
        elif command_steering == 1900:
            command_num = 3
            print("left")
        elif command_steering == 1101:
            command_num = 4
            print("right")
        elif command_velocity == 1543:
            command_num = 5
            print("slow")
        elif command_velocity == 1561:
            default_velocity = 1545
            command_num = 6
            print('fast')
        elif command_velocity == 1506:
            command_num = 7
            print("red_light!")
        elif command_velocity == 1562:
            default_velocity = 1555
            command_num = 8
            print("green_light!go!")
        elif command_velocity == 1573:
            default_velocity = 1555
            command_num = 2
            print("Change Road!")
        elif command_steering == 1879:
            command_num = 9
            print("attention!precipice!")
        elif command_velocity == 1509:
            default_velocity = 1500
            default_steering = 1500
            command_num = 10
            print("END!")

        if command_num == 1:
            if flag_crossing == 0:
                flag_crossing = 1
                car_control(command_velocity, default_steering)
                flag_left = 0
                flag_right = 0
                time.sleep(1)
                car_control(default_velocity, default_steering)
                command_num = 2
            else:
                command_num = 2
        elif command_num in (3, 4):
            if command_num == 3:
                flag_left = 1
            if command_num == 4:
                flag_right = 1
            car_control(default_velocity, command_steering)
            time.sleep(1)
            flag_roundabout = True
            roundabout_start_time = datetime.now()
            command_num = 2
        elif command_num == 5:
            command_num = 0
            default_velocity = 1545
            default_steering = 1500
        elif command_num == 6:
            command_num = 2
        elif command_num == 7:
            if flag_red == 0:
                flag_red = 1
                car_control(default_velocity, 1300)
                time.sleep(0.2)
                car_control(default_velocity, 1500)
                time.sleep(0.6)
                car_control(default_velocity, 1900)
                time.sleep(0.5)
                car_control(1505, default_steering)
        elif command_num == 8:
            if flag_green == 0:
                flag_green = 1
                car_control(1545, 1100)
                time.sleep(0.2)
                command_num = 2
                flag_park = 1
            else:
                command_num = 2
        elif command_num == 9:
            if flag == 0:
                car_control(default_velocity, command_steering)
                time.sleep(1.2)
                car_control(default_velocity, 1500)
                time.sleep(0.4)
                car_control(default_velocity, 1100)
                time.sleep(0.7)
                car_control(default_velocity, 1500)
                time.sleep(0.4)
                car_control(default_velocity, 1900)
                time.sleep(1.2)
                flag_roundabout_2 = True
                roundabout_start_time_2 = datetime.now()
                command_num = 2
                flag = 1
            else:
                command_num = 0
        elif command_num == 10:
            car_control(default_velocity, 1300)

        if command_num == 2:
            command_num = 0
            default_velocity = 1545
            default_steering = 1500


    if __name__ == "__main__":
        try:
            serial_connection = serial.Serial(car_port, 9600)
        except Exception as e:
            print(f"串口错误: {e}")
            sys.exit(-1)
        time.sleep(1)

        lane_process = Process(target=follow_lane)
        sign_process = Process(target=recognize_signs)
        lane_process.start()
        sign_process.start()

        try:
            lane_process.join()
            sign_process.join()
        except KeyboardInterrupt:
            print("由于键盘中断，正在退出...")
            car_control(1500, 1500)
        finally:
            sys.exit(0)

> <font size=5>代码总体梗概</font>

---

<font size=5>&emsp;&emsp;让电脑开2个线程，一个跑循迹，一个跑路牌识别。识别到路牌后发送给队列（一个栈）命令代号，程序在无命令代号的时候听循迹的话跑路线，有命令代号的时候按命令代号对应的操作执行命令。之后退出命令号，继续由循迹操作车辆。相当于一个又一个中断。在此基础上，我在左右转后进入人行横道前加入了定时器，在时间到后转出弯道。还在最后加了定时器，为了冲线后自己停车。大概情况就是这样，主要是细枝末节很多，调试了数月，增加了大量代码。\
</font>