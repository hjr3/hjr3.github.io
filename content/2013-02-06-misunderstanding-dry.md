+++
title = "Misunderstanding DRY"
path = "/2013/02/06/misunderstanding-dry.html"
+++

I think the Object Oriented (OO) principle of Don't Repeat Yourself (DRY) is often misunderstood. In particular, the word "repeat" is troublesome. It has nothing to do with minimizing the amount of code you write. It is also not about merging similar methods together into a super-method. The DRY principle is about preserving a single source of truth in a system. When a there are multiple sources of truth in a system we have to write more code to manually keep all the truths in sync with each other. This often leads to unintended consequences to a part of the system when a change is made to a different part of the system. We are left to look through the code looking for these unintended consequences and become increasingly reluctant to change. Properly applying the DRY principle protects us from these unintended consequences and can make our code much more accepting of change.

To those less experienced, DRYing up some parts of the code may not seem like a waste of time in the present. I want to use some code samples in an attempt to prove that fixing even simple DRY violations can be very helpful. The following class is a simple `Car` class.  It starts out with a single method `currentSpeed()` which returns the current speed of the `Car` instance. In a real class, there would be more implementation detail. For now we are just concerned with the speed of the `Car` class. We will change the `Car` and use the DRY principles to help us design good code.

    class Car
    {
        protected $speed;
    
        public function currentSpeed() { return $this->speed; }
    }

At this point the class seems pretty reasonable. The `$speed` member variable stores the speed of the `Car` instance. The `currentSpeed()` method simply returns the speed. Now let us pretend that we need to add the logic for cruise control. A basic cruise control system is made up of four operations: toggle, set, cancel and resume. The toggle operation turns the cruise control on and off. The set operation will determine the speed of the `Car` object and maintain that speed. The cancel operation instructs the cruise control system to stop maintaining the set speed. The resume operation signals the cruise control system to accelerate to the set speed and then maintain that speed. The toggle operation is uninteresting, so let's start implementing the set operation. We might do something like this:

    class Car
    {
        protected $speed;
        protected $cruisingSpeed;
    
        public function currentSpeed() { return $this->speed; }
    
        public function cruiseControlSet() {
            $this->cruisingSpeed = $this->speed;
        }
    }

This is a simple change, but we actually just violated the DRY principle. The `cruiseControlSet()` method should not have direct access to the `$speed` member variable. Good OO design focuses on passing around message (or methods) and not data. The `$speed` member variable is data. We should use the `currentSpeed()` method to _ask_ for the speed. I made a special point to use the word _ask_ in the previous sentence. Our current implementation is not asking for anything. It knows _how_ the `$speed` data is stored within `Car` class. What is the big deal though, right? It is obvious by looking at this code that two different methods are accessing `$speed`. If we are going to change how `$speed` works later on, we can deal with it then. YAGNI bro!

Trying to predict the future is sure way to make your code design overly complex. The principle of You Aren't Gonna Need It (YAGNI) addresses this concern. However, we have already established a pattern here. More than one method needs to know the speed of the `Car` object. There is a good chance that more methods will need to know the speed as well. It is also important to consider the cost of making a change. In this case, the cost of changing `cruiseControlSet()` to use the `currentSpeed()` method instead of directly accessing `$speed` is very low. When the cost is low, err on the side of good OO design in an attempt to make change easy. Do it even if you are sure that `$speed` will never change.

Something else starts to become apparent as we add more of the cruise control functionality to the `Car` class. Let's add the other methods and see if we can spot it.

    class Car
    {
        protected $speed;
        protected $cruisingSpeed;
    
        public function currentSpeed() { return $this->speed; }
        public function cruisingSpeed() { return $this->cruisingSpeed; }
    
        public function cruiseControlToggle() { ... }
    
        public function cruiseControlSet() {
            $this->cruisingSpeed = $this->currentSpeed();
        }
    
        public function cruiseControlCancel() { ... }
    
        public function cruiseControlResume() {
            if ($this->currentSpeed() != $this->cruisingSpeed()) {
                ...
            }
        }
    }

As we are adding the cruise control functionality to the `Car` class something starts to feel wrong. The class is getting large in a hurry. Also, our tests may be getting harder to setup. This functionality is screaming to be refactored into a separate class. Another hint is that we started using a common method prefix of `cruiseControl`. Whenever this happens, we should really consider if this functionality is part of this class. Let's move all the `cruiseControl*()` methods and the `cruisingSpeed()` method into another class. Watch closely how the DRY principle helps us minimize the amount of changes we make in this refactor.

    class Car
    {
        protected $speed;
        protected $cruisingSpeed;
    
        public function currentSpeed() { return $this->speed; }
    }
    
    class CruiseControl 
    {
        public function cruisingSpeed() { return $this->cruisingSpeed; }
    
        public function toggle() { ... }
    
        public function set() {
            $this->cruisingSpeed = $this->currentSpeed();
        }
    
        public function cancel() { ... }
    
        public function resume() {
            if ($this->currentSpeed() != $this->cruisingSpeed()) {
                ...
            }
        }
    
        public function __construct(Car $car)
        {
            $this->car = $car;
        }
    
        protected function currentSpeed()
        {
            return $this->car->currentSpeed();
        }
    }

Notice how the methods that implement our cruise control operations are still using the `currentSpeed()` method. They did not have to change because we just added a protected method to get the current speed of the car. We hide away the knowledge of where the speed is coming from as these methods are not concerned with that specific knowledge. We are able to do this because the cruise control is given access to the public interface of the `Car` class and can then determine the speed. This forces the cruise control system to _ask_ the `Car` class to do things. For example, the cruise control system no longer has the potential to change the `$speed` of the `Car` class. If it wants to change speeds, it must _ask_ a `Car` object to accelerate or decelerate.

I hope I have shown the benefits of DRYing up code, even when the DRY violations appear to be harmless. You may still have some reservations about the benefits of the DRY principle. I encourage you to put the those reservations aside for a period of time and adhere to those principles. Chances are you will notice an increase in the quality of design.
