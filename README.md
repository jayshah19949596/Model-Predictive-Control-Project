# CarND-MPC-Project
----
This repository contains C++ code for implementation of Model Predictive Controller. MPC is used to derive throttle, brake and steering angle actuators for a car to drive around a circular track. This task was implemented to partially fulfill Term-II goals of Udacity's self driving car nanodegree program.


## Background
----
A critical module in the working of any robotic system is the control module. Control module defines the action, which the robotic system performs in order to achieve a task. These actions can vary on type of the system and type of the task. For e.g.: A simple mixer grinder's control module only controls the speed of rotating motor. A little more complex system such as a Remote Controlled (RC) car needs a control module to move forward, backward and turn. Highly complex systems such as prosthetic arms, self driving cars, product manufacturing factory units require control modules for multiple tasks at the same time.

One of the basic implementation of a control system is a Proportional (P), Differential (D), Integral (I), together, a PID controller. PID controller is the most popular controller and is used in applications across domains. But PID controller cannot be used for controlling complex systems such as self-driving vehicles as one needs to guarantee a smooth and safe journey and not just navigation from one point to another.

A more sophisticated class of controllers is the Model Predictive Controller (MPC). MPC is built taking into consideration the motion model of the system. Hence, this controller adapts to any sort of secondary ask along with the primary ask. For e.g., as mentioned earlier, MPC can be used not only to perform the primary task of navigation but also to ensure a smooth ride of a self-driving vehicle. This is possible as one can start with a simple motion model and then easily add new parameters to the cost function. Also, being more sophisticated, MPC can also be used to model different uncertainties and external environmental factors into the motion model of the system.


## Working of Model Predictive Controller
----
MPC is a non-linear system which generates optimized parameters with the constraint of obtaining minimal value of cost function. Here, cost function refers to the problem built by taking into consideration the model of the system, range of inputs, limitations on outputs and/or the effect of external factors acting on the system. 

A typical cost function in case of a self-driving vehicle will have following constraints:

  1. The cross track error (cte), i.e. the distance of vehicle from the center of the lane must be minimal.
  2. The heading direction of the vehicle must be close to perpendicular to the lane. Error PSI (epsi) is the error in heading direction
  3. Oscillations while riding the vehicle must be minimal, or one would feel sea sick. This takes into account the change in heading direction due to turns and also the change in speed due to acceleration/braking.
  4. The vehicle must drive safely and should not exceed the speed limit. In contrast, the vehicle must also not drive too slow as one wouldn't reach any place.
  
These constraints are merged to form a cost function. MPC tries to reduce the cost function and come up with the values of actuation inputs to the vehicle, which can be the steering angle of the wheel and/or the throttle/brake magnitude.


## Project Goal
----
In this project, MPC was implemented to drive a car around a circular track having sharp left and right turns. A good solution would help the car stay in the center portion of the lane and take smooth left and right turns without touching or running over the edges of the lane (considered as risky in case humans were travelling in such a car). Also, the final implementation had to be tested with speeds as high as 100mph and note down the behavior of the car.

## Project Implementation
----
Simulation of a circular track was achieved in the [Udacity's self driving car simulator](https://github.com/udacity/self-driving-car-sim/releases). While MPC was implmented in C++, the simulator communicated to C++ code with the help of [uWebSockets library](https://github.com/uNetworking/uWebSockets). Following parameters were received from the simulator for each communication ping:  

  1. List of waypoints with x and y coordinates of each point in global map system. These waypoints represent the suggested trajectory for the car from current position to few distance ahead of it. Such type of information is usually received from a [Path Planning Module](https://en.wikipedia.org/wiki/Motion_planning) implemented in self-driving vehicles. The path planning module generates reference trajectory for a section of the total journey based on the current location of the vehicle.
  
  2. The current position and heading of car in global map system.
  
  3. The current steering angle and throttle applied to the car.
  
  4. The current speed of the car.  
  

The final implementation consisted of following "major steps:

### 1. Changing the Co-Ordinate origin to Car's Co-ordinate system:
  
  - In this step, the coordinates of waypoints were transformed from global map system to vehicle's coordinate system. 
  - This was done to ensure the calculations of cte and epsi were less complex and involved less calculations. 
  - By doing this the x and y co-ordiante poitns are started from (0, 0) and the orientation of the car is 0 angle
  - This reduces the calucaltions 


### 2. Generating reference trajectory from waypoints:
  
  - The transformed coordinates of waypoints were then used to create a smooth curved trajectory, which will act as a reference for motion of the car. This is done by using [polynomial fitting or regression technique](https://en.wikipedia.org/wiki/Polynomial_regression)
  - In this technique, discrete points are fitted to a polynomial of desired degree in order to obtain the relation function between x and y coordinates, also known as the curve equation.
  - In this project, the waypoints coordinates were fitted to a **3rd degree** polynomial. A smooth curve was drawn inside the scene in the simulator shown by yellow line in the screen cap below
  - The polynomial was fitted to the points by using [JuliaMath polyfit function](https://github.com/JuliaMath/Polynomials.jl/blob/master/src/Polynomials.jl) 
  - The function returned the coefficients of the polynomial line function 
  - This co-efficients are used to geenrate the reference tracjectory which gives us set of continuous points
  
  
### 3. Calculation of CTE and EPSI:
  
  - As a result of transformation of data in step 1, the calculation of cte was a linear function and the calculation of epsi involved taking arctangent of the first order coefficient of fitting polynomial 
  
  
### 4. Definition of motion model, control/actuator inputs and state update equations:
  
  - The motion model in this project was based on the [Kinematic equations of motion](https://en.wikipedia.org/wiki/Equations_of_motion#Kinematic_quantities) in 2 dimensions. 
    
  - The state consists of following parameters:  
  
     1. The x coordinate of position of car in vehicle's coordinate system (x)
     2. The y coordinate of position of car in vehicle's coordinate system (y)
     3. The heading direction of car in vehicle's coordinate system (psi)
     4. The magnitude of velocity of car (v)
    
  - The actuator inputs used to control the car are given below:  
  
     1. The magnitude of steering (delta). This was limited to [-25, 25] as per the angle specification of simulator.
     2. The magnitude of throttle (a). This was limited to [-1, 1] as per the throttle/brake specification of simulator.   
     
  - The state update equations are then given by:
    
[image1]: ./img_resources/state_update_eqn.png "Kinematic State update equations"
![Kinematic State update equations][image1]


### 5. Definition of time step length (N) and duration between time steps (dt):
  
  - Given the reference trajectory from polynomial fit of waypoints and the motion model of the car, MPC estimates the value of actuator inputs for current time step and few time steps later
  - This estimate is used to predict the actuator inputs to the car ahead of time 
  - This process of estimate generation is tunable with the use of N and dt
  - Higher value of N ensures more number of estimates while higher value of dt ensures the estimates are closer in time.
  - I chose N value to 10 and dt value to 0.1

### 6. Definition of cost function for MPC:
  
  - The last step in the implementation is to define the cost function for MPC. MPC solver, implemented using [Ipopt](https://projects.coin-or.org/Ipopt) and [Cppad](https://www.coin-or.org/CppAD/) library generated actuator values while arriving at the minimal value of cost function
  - Key elements and features of the cost function are given below:
    
    1. Highest weight for calculated CTE and EPSI. This was to ensure the car stays in the middle of lane and head in desired direction
    2. Reduce high initial values of control inputs (delta and a) to ensure there is no jerk in motion of the car
    3. Minimize the change in values of control inputs (delta and a) in order to minimize oscillations in the motion
    4. Minimize speeds at higher steering angle and minimize steering at higher speeds. This was to ensure the car took smooth turns by reducing the speed while it reached maximum possible speed on straight ahead path
     
     
### 7. Incorporating effect of latency in the control actuations:
  
  - In real world systems as complex as commercial jet planes, there exists certain amount of delay in time between the actuation of control and its effect on the motion
  - To achieve an implementation close to real world scenario, a latency of 100ms was introduced in the simulator. 
  - This delay caused control actuations to reach the car in later of time 
  - To take into account the effect of this latency, actuation commands from previous timestep is used which solved the problem


# Project Output
----
- MPC used to derive the steering angles and throttle/brake for a car moving on a circular track was implemented successfully
- The car could stay close to the center of the lane and take smooth left and right turns along its path 
- This was achieved in spite of presence of latency in the system
- Detailed insight into features of the simulator and implementation is demonstrated in this [MPC demo video](https://youtu.be/TVXn7eSOWJw).


## Dependencies
----
* cmake >= 3.5
 * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1(mac, linux), 3.81(Windows)
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
  * Windows: [Click here for installation instructions](http://gnuwin32.sourceforge.net/packages/make.htm)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools]((https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)
* [uWebSockets](https://github.com/uWebSockets/uWebSockets)
  * Run either `install-mac.sh` or `install-ubuntu.sh`.
  * If you install from source, checkout to commit `e94b6e1`, i.e.
    ```
    git clone https://github.com/uWebSockets/uWebSockets
    cd uWebSockets
    git checkout e94b6e1
    ```
    Some function signatures have changed in v0.14.x. See [this PR](https://github.com/udacity/CarND-MPC-Project/pull/3) for more details.

* **Ipopt and CppAD:** Please refer to [this document](https://github.com/udacity/CarND-MPC-Project/blob/master/install_Ipopt_CppAD.md) for installation instructions.
* [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page). This is already part of the repo so you shouldn't have to worry about it.
* Simulator. You can download these from the [releases tab](https://github.com/udacity/self-driving-car-sim/releases).
* Not a dependency but read the [DATA.md](./DATA.md) for a description of the data sent back from the simulator.


## Basic Build Instructions
----
1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./mpc`.

## Tips
----
1. It's recommended to test the MPC on basic examples to see if your implementation behaves as desired. One possible example
is the vehicle starting offset of a straight line (reference). If the MPC implementation is correct, after some number of timesteps
(not too many) it should find and track the reference line.
2. The `lake_track_waypoints.csv` file has the waypoints of the lake track. You could use this to fit polynomials and points and see of how well your model tracks curve. NOTE: This file might be not completely in sync with the simulator so your solution should NOT depend on it.
3. For visualization this C++ [matplotlib wrapper](https://github.com/lava/matplotlib-cpp) could be helpful.)
4.  Tips for setting up your environment are available [here](https://classroom.udacity.com/nanodegrees/nd013/parts/40f38239-66b6-46ec-ae68-03afd8a601c8/modules/0949fca6-b379-42af-a919-ee50aa304e6a/lessons/f758c44c-5e40-4e01-93b5-1a82aa4e044f/concepts/23d376c7-0195-4276-bdf0-e02f1f3c665d)
5. **VM Latency:** Some students have reported differences in behavior using VM's ostensibly a result of latency.  Please let us know if issues arise as a result of a VM environment.

## Editor Settings
----
We've purposefully kept editor configuration files out of this repo in order to
keep it as simple and environment agnostic as possible. However, we recommend
using the following settings:

* indent using spaces
* set tab width to 2 spaces (keeps the matrices in source code aligned)

## Code Style
----
Please (do your best to) stick to [Google's C++ style guide](https://google.github.io/styleguide/cppguide.html).

## Project Instructions and Rubric
----
Note: regardless of the changes you make, your project must be buildable using
cmake and make!

More information is only accessible by people who are already enrolled in Term 2
of CarND. If you are enrolled, see [the project page](https://classroom.udacity.com/nanodegrees/nd013/parts/40f38239-66b6-46ec-ae68-03afd8a601c8/modules/f1820894-8322-4bb3-81aa-b26b3c6dcbaf/lessons/b1ff3be0-c904-438e-aad3-2b5379f0e0c3/concepts/1a2255a0-e23c-44cf-8d41-39b8a3c8264a)
for instructions and the project rubric.

## Hints!
----
* You don't have to follow this directory structure, but if you do, your work
  will span all of the .cpp files here. Keep an eye out for TODOs.

## Call for IDE Profiles Pull Requests
----
Help your fellow students!

We decided to create Makefiles with cmake to keep this project as platform
agnostic as possible. Similarly, we omitted IDE profiles in order to we ensure
that students don't feel pressured to use one IDE or another.

However! I'd love to help people get up and running with their IDEs of choice.
If you've created a profile for an IDE that you think other students would
appreciate, we'd love to have you add the requisite profile files and
instructions to ide_profiles/. For example if you wanted to add a VS Code
profile, you'd add:

* /ide_profiles/vscode/.vscode
* /ide_profiles/vscode/README.md

The README should explain what the profile does, how to take advantage of it,
and how to install it.

Frankly, I've never been involved in a project with multiple IDE profiles
before. I believe the best way to handle this would be to keep them out of the
repo root to avoid clutter. My expectation is that most profiles will include
instructions to copy files to a new location to get picked up by the IDE, but
that's just a guess.

One last note here: regardless of the IDE used, every submitted project must
still be compilable with cmake and make.


## How to write a README
----
A well written README file can enhance your project and portfolio.  Develop your abilities to create professional README files by completing [this free course](https://www.udacity.com/course/writing-readmes--ud777).

