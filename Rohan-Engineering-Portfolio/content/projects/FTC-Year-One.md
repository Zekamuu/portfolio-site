---
title: "FTC Robotics, Year 1"
date: 2022-10-01
description: "What I built in the first year on the team"
image: "/img/bot-screenshot.jpg"  # Specific to Beautiful Hugo to show a preview image
---
Objective
Primary: Quickly pick up cones and score them on low, medium, and high junctions.
Autonomous: Score pre-loaded cone, detect randomized signal sleeve, and park reliably.
Driver-Controlled: Enable driver to score as many cones as possible in 2 minutes, prioritizing speed and ease of use.

Design Process
Identified the need for a holonomic drivetrain for agility, leading to the selection of a Mecanum Drive.
Determined that a multi-stage Cascading Linear Lift (Viper Slide) would provide the necessary reach and stability to score on the high junction

Prototyping:
Hardware: Built a 4-motor Mecanum chassis, a 3-stage lift actuated by a single motor with Dyneema rigging, and a 3D-printed two-servo claw with foam inserts for grip.

Iteration:
Autonomous: Implemented TensorFlow Lite with a custom-trained model to detect three possible AprilTag signals. Wrote encoder-based movement paths to strafe and park based on the vision result.
Tele-Op: Refined lift controls from simple manual power to a precise, encoder-based RUN\_TO\_POSITION system. This allowed the driver to set a target height, and the robot's PID controller would move to and hold that position automatically

Hardware Abstraction
BaseOpMode is responsible for mapping all the robot's hardware components from the configuration file to Java objects. This centralized approach ensures consistency across all OpModes.
Mecanum Drivetrain: Four DcMotorEx motors (leftFront, rightFront, leftBack, rightBack) are configured for a mecanum drive, allowing for holonomic movement (strafing). Motor directions are reversed as needed to ensure intuitive forward motion.
```
rightFront.setDirection(DcMotor.Direction.REVERSE);
leftFront.setDirection(DcMotor.Direction.REVERSE);
leftBack.setDirection(DcMotor.Direction.REVERSE);
rightBack.setDirection(DcMotor.Direction.REVERSE);
```


Vision System: TensorFlow Object Detection

The robot uses the TensorFlow Object Detection API to identify the randomized  signal on the sleeve.
Model and Labels: A custom-trained TensorFlow Lite model is loaded from the robot controller's storage. This model is trained to recognize three custom labels: "s1", "s4", and "s5", which correspond to the three possible images on the signal sleeve.

```
private static final String TFOD\_MODEL\_FILE = "/sdcard/FIRST/tflitemodels/model\_20221023\_141235.tflite";
private static final String[] LABELS = {
  "s1",
  "s4",
  "s5"
};
```
Detection Logic: After the OpMode starts, the robot enters a loop, scanning for recognitions from the TensorFlow engine. It iterates through detected objects, summing the confidence scores for each of the three possible labels. The label with the highest cumulative confidence above a threshold (0.8f) is selected as the identified signal (selectedSymbol).
```
float maxValue = 0.8f;
int i = 0;
for (float confidence : symbolConfidences){
    if (confidence > maxValue){
        maxValue = confidence;
        selectedSymbol = i;
    }    
    i += 1;
}
```

Autonomous Pathing
The autonomous routine is a sequence of encoder-based movements.
Initial Movement: The robot drives forward a set distance (1100 encoder ticks) to position itself for scanning the signal sleeve.
Signal Detection: The TensorFlow detection loop runs until a signal is confidently identified.
Parking: Based on the value of selectedSymbol (0, 1, or 2), the robot executes a specific movement to park in the correct zone. The code implements paths for symbols 0 and 2, which involve strafing left or right.
```
switch(selectedSymbol){
    case 0: // Strafe Left
        leftFront.setTargetPosition(0);
        rightFront.setTargetPosition(2200);
        leftBack.setTargetPosition(2200);
        rightBack.setTargetPosition(0);
        break;
    case 2: // Strafe Right
        leftFront.setTargetPosition(2200);
        rightFront.setTargetPosition(0);
        leftBack.setTargetPosition(0);
        rightBack.setTargetPosition(2200);
        break;
}
```
This routine reliably parks the robot, fulfilling a key requirement of the autonomous period. Scoring the pre-loaded cone would be the next logical step to implement in this sequence.

Drivetrain Control
The robot uses a field-oriented mecanum drive, but the current implementation in TeleOpMain.java uses a simpler robot-oriented control scheme for rotation.
Mecanum Kinematics: The left joystick's X and Y values are used to calculate a movement vector (magnitude and direction). These are translated into individual power levels for each of the four mecanum wheels, allowing for omnidirectional movement.
Rotation: The right joystick's X value (oldRotation) is added to the wheel power calculations to make the robot turn. While there is code present for IMU-based heading correction, it appears the simpler, direct joystick rotation is currently in use for driving. This prioritizes direct driver feedback
```
float leftFrontPower = magnitude * (float)Math.sin(direction + Math.PI / 4) + oldRotation;
float leftBackPower = magnitude * (float)Math.sin(direction - Math.PI / 4) + oldRotation;
float rightFrontPower = magnitude * (float)Math.sin(direction - Math.PI / 4) - oldRotation;
float rightBackPower = magnitude * (float)Math.sin(direction + Math.PI / 4) - oldRotation;
```

Lift
The D-pad (dpad\_up, dpad\_down) is used to increment or decrement a targetPosition for the lift motors. The motors, running in RUN\_TO\_POSITION mode, automatically move to and hold this precise height. This is ideal for quickly selecting the low, medium, or high junction heights.
```
if(motorUp){
    targetPosition += liftSpeed;
} else if (motorDown){
    targetPosition -= liftSpeed;
}
// Clamp targetPosition to min/maxPosition
targetPosition = (int)
    Math.min(maxPosition,
    Math.max(minPosition, targetPosition)
    );
motorOne.setTargetPosition(targetPosition);
```

Claw
The left trigger ```(gamepad1.left_trigger)```, an analog input, directly controls the position of the two claw servos. This allows the driver to have fine-grained control over how tightly the cone is gripped
```
float clawOpen = gamepad1.left_trigger;
clawServoRight.setPosition(clawOpen);

clawServoLeft.setPosition(1-clawOpen);
```
The project includes several simple test OpModes (```ClawServoTest.java```, ```LiftMotorTest.java```, ```TankDrive.java```, ```ArcadeDrive.java```) that were used during development to isolate and validate individual hardware components before integrating them into the main OpModes. 

Outcome 

A competitive robot with a robust and modular software architecture.
>The autonomous mode reliably detected the signal sleeve and parked in the correct zone
>The driver-controlled mode was highly effective, featuring precise lift controls and intuitive operation that enabled efficient cone scoring cycles.

