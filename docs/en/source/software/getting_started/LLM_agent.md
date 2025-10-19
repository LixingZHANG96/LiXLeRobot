## LLM Agent Control

Imagine telling the robot "go cleanup my kitchen" and watching him doing so. This tutorial will show you how you can make your XLeRobot fully autonomous, self-decision making machine by providing it with LLM agent. An agent that, by seeing world through camera and hearing your voice commands, be able to move the robot and (in the future) manipulate objects using VLA policies.

## Getting Started

To create our agent, we will use the [RoboCrew](https://github.com/Grigorij-Dudnik/RoboCrew) library - one specially designed for embodied agents. On your control device (Raspberry Pi or laptop) create a new virtual envinronment and install it with:

```bash
pip install robocrew
```

Next, create the python script to control your robot. Let's start by creating a simple agent that performs just one hardcoded task and finishes. 

First, let's create tools for agent to control a wheel movement:

```python
from robocrew.core.LLMAgent import LLMAgent
from robocrew.core.tools import finish_task
from robocrew.robots.XLeRobot.tools import create_move_forward, create_turn_left, create_turn_right
from robocrew.robots.XLeRobot.wheel_controls import XLeRobotWheels

# Set up wheels
sdk = XLeRobotWheels.connect_serial("/dev/ttyUSB0")     # provide the right arm usb port - the arm connected to wheels
wheel_controller = XLeRobotWheels(sdk)

# Create movement tools
move_forward = create_move_forward(wheel_controller)
turn_left = create_turn_left(wheel_controller)
turn_right = create_turn_right(wheel_controller)
```

in place of `"/dev/ttyUSB0"` you should provide the USB port name of your right arm (one connected to wheels).

ToDo: add here a short instruction on how to find USB port names and how to bind them to specific devices to avoid usb name swapping.

Next, let's initialize agent itself:

```python
# Create agent
agent = LLMAgent(
    model="google_genai:gemini-robotics-er-1.5-preview",
    tools=[move_forward, turn_left, turn_right, finish_task],
    main_camera_usb_port="/dev/video0",     # provide usb port main camera connected to
    camera_fov=110,
)
agent.task = "Find kitchen in my house and go there."

agent.go()
```

In place of `"/dev/video0"` you need to provide USB port name of your main camera.

In the code above we initialized our agent with the maneuver tools we created earlier. You can provide any model in LangChain notation. Next, we hard-coding a task for our agent and running it.

Before going to LLM, camera image is specially augmented to make it easier for robot predict his angle of rotation. Special scale of angles is added to the camera, plus right/left markers - LLM likes to mismatch directions. That's why we specifing `camera_fov` parameter - provide here your main camera horizontal angle of view, to make drawn scale precise.

<div style="text-align: center; font-style: italic">
  <img src="https://github.com/user-attachments/assets/31f063b9-2463-4ba5-b5da-311f16788576" width="60%">
  <p>That's how your robot sees the world.</p>
</div>

Also, create the `.env` file with parameter `GOOGLE_API_KEY=<your gemini api key here>` inside to connect to your model.

Now run the code and watch if your XLeRobot will be able to find that kitchen - then it will finish its work by calling `finish_task` tool!

Complete code is here:

```python
from robocrew.core.LLMAgent import LLMAgent
from robocrew.core.tools import finish_task
from robocrew.robots.XLeRobot.tools import create_move_forward, create_turn_left, create_turn_right
from robocrew.robots.XLeRobot.wheel_controls import XLeRobotWheels

# Set up wheels
sdk = XLeRobotWheels.connect_serial("/dev/ttyUSB0")     # provide the right arm usb port - the arm connected to wheels
wheel_controller = XLeRobotWheels(sdk)

# Create movement tools
move_forward = create_move_forward(wheel_controller)
turn_left = create_turn_left(wheel_controller)
turn_right = create_turn_right(wheel_controller)

# Create agent
agent = LLMAgent(
    model="google_genai:gemini-robotics-er-1.5-preview",
    tools=[move_forward, turn_left, turn_right, finish_task],
    main_camera_usb_port="/dev/video0",     # provide usb port main camera connected to
    camera_fov=110,
)
agent.task = "Find kitchen in my house and go there."

agent.go() 
```

## Voice-conrolled agent

Now that we have managed to run our simple agent, let's add him an ability to listen to our voice commands through the microphone.

First, we need to install Portaudio, to enable our control device to hear:

```bash
sudo apt install portaudio19-dev
```

Tools initialization remains the same as in previous example - the only change here is we no longer need the `finish_task` tool anymore, as our agent will be listen for our tasks and operate in endless (untill we shut down the script).

What changes is the agent initialization:

```python
agent = LLMAgent(
    model="google_genai:gemini-robotics-er-1.5-preview",
    tools=[move_forward, turn_left, turn_right],
    main_camera_usb_port="/dev/video0",  # provide usb port main camera connected to
    sounddevice_index=0,  # Your mic device
    wakeword="robot",  # The robot listens for this word in your speech
    history_len=4,
)

agent.go()
```

As you can see, we need to provide index of our soundcard with microphone. We can also set up our wakeword (default is "robot") - when robot hears that word in your sentence, it threats it as its new task; otherwise ignores it.

ToDo: short instruction on how to find soundcard index and make it constant.

We can also set up `history_len` - how many of last movements robot should keep in the memory, to avoid memory overflow.

Run the code and ask the robot to do something for you!

Complete code is here:

```python
from robocrew.core.LLMAgent import LLMAgent
from robocrew.robots.XLeRobot.tools import create_move_forward, create_turn_left, create_turn_right
from robocrew.robots.XLeRobot.wheel_controls import XLeRobotWheels

# Set up wheels
sdk = XLeRobotWheels.connect_serial("/dev/ttyUSB0")     # provide the right arm usb port - the arm connected to wheels
wheel_controller = XLeRobotWheels(sdk)

# Create movement tools
move_forward = create_move_forward(wheel_controller)
turn_left = create_turn_left(wheel_controller)
turn_right = create_turn_right(wheel_controller)

# Create agent
agent = LLMAgent(
    model="google_genai:gemini-robotics-er-1.5-preview",
    tools=[move_forward, turn_left, turn_right],
    main_camera_usb_port="/dev/video0",  # provide usb port main camera connected to
    sounddevice_index=0,  # Your mic device
    wakeword="robot",  # The robot listens for this word in your speech
    history_len=4,
)

agent.go() 
```