# spaghetti manual

For reference, please download/ clone whole repo and run ```make doc```/

This page demonstrates usage through several showcases

## Showcase 1: Hello World for FSM

Once you have downloaded the file ```spaghetti.hpp```, you are able to make the [turnstyle (WP link)](https://en.wikipedia.org/wiki/Finite-state_machine#Example:_coin-operated_turnstile) example run, which is the "Hello World" for Finite State Machines.

First, create enums for states and events:

```C++
#include "spaghetti.hpp"

enum States { st_Locked, st_Unlocked, NB_STATES };
enum Events { ev_Push, ev_Coin, NB_EVENTS };
```

As you can see, the requirement here is that they **must** have as last element
```NB_STATES``` and ```NB_EVENTS```, respectively.
For the states, the first one (having value 0) will be the initial state.

Then, create the FSM data type:
```C++
SPAG_DECLARE_FSM_TYPE_NOTIMER( fsm_t, States, Events, bool );
```
This means you declare the type ```fsm_t```, using ```States``` and ```Events```, with callback functions having a bool as an argument.
(Alternatively, you could also use a ```typedef```, but lets say this is easier at present.)

Now, you can instanciate the fsm:

```C++
int main()
{
	fsm_t fsm;
```

Now, you need to configure your FSM, that is define what event in what state will trigger switching to what state.
With this simple example, you just do:
```C++
	fsm.assignTransition( st_Locked,   ev_Coin, st_Unlocked );
	fsm.assignTransition( st_Unlocked, ev_Push, st_Locked );
```
This means:
- if a coin is inserted while in "locked" state, then switch to state "unlocked".
- if somebody pushes the gate while in "unlocked" state, then switch to state "locked".

Ok, and also tell the FSM what is to be done when a state gets triggered.
This is done by providing a callback function.
In this case it is the same for all, but you can have a different function for all the states.
You don't even need to provide a callback function, it is always optional:
some states can have one while others won't.
The only constraint is that they must have the same signature.

```C++
	fsm.assignCallback( st_Locked,   cb_func, true );
	fsm.assignCallback( st_Unlocked, cb_func, false );
```

Alternatively (and that is useful if you have a lot of states), you can provide the callback function for all the states, and separately set the argument value:

```C++
	fsm.assignGlobalCallback( cb_func );
	fsm.assignCallbackValue( st_Unlocked, false );
	fsm.assignCallbackValue( st_Locked,   true );
```

And of course you must provide that callback function:

```C++
void cb_func( bool b )
{
	if( b )
		std::cout << "State: Locked\n";
	else
		std::cout << "State: Unlocked\n";
}
```
Done for configuration.
Now, how to run this. (Here, you don't really need to because there are no timeout, but its recommended to do so, to avoid forgetting it afterwards. Just call ```start()``` ).

Here the events will be triggered by the keyboard, so lets do this:
```C++
fsm.start();
do
{
	char key;
	std::cout << "Enter command: ";
	std::cin >> key;
	switch( key )
	{
		case 'A':
			std::cout << "Event: push\n";
			fsm.processEvent( ev_Push );
		break;

		case 'B':
			std::cout << "Event: coin\n";
			fsm.processEvent( ev_Coin );
		break;
	}
}
while( 1 );
```

That's it!
All of this is given as a sample program,
see file ```src/turnstyle_1.cpp``` and/or just clone repo end enter
```make demo -j4``` followed by ```bin/turnstyle_1```.


## Showcase 2: let's use a timer

Lets consider another situation: a traffic light going automatically through the three states: Red, Green, Orange.
You need to provide a Timer class that can be used by the FSM.
This class needs to provide these three functions:
- ```timerInit()```: initialize the timer
- ```timerStart()```: start a timeout
- ```timerCancel()```: cancel the timer

Any timing class can be used, in the provided sample ```src/traffic_lights.cpp``` we demonstrate the use of [boost::asio](http://www.boost.org/doc/libs/release/libs/asio/):
```C++
template<typename ST, typename EV, typename CBA>
struct AsioWrapper
{
	boost::asio::io_service io_service;
	std::unique_ptr<boost::asio::deadline_timer> ptimer;

	AsioWrapper()
	{
		ptimer = std::unique_ptr<boost::asio::deadline_timer>( new boost::asio::deadline_timer(io_service) );
	}
...
```
(see full code for details)
The three type parameters are the states, the events, and the type of the argument of the callback function.

Once you have declared this class, the declaration of the data type of the FSM will be done with the second form of the macro:
```C++
SPAG_DECLARE_FSM_TYPE( fsm_t, States, Events, AsioWrapper, bool );
```
The configuration step will go as follows (assuming the states are names ST_RED, ST_GREEN, ST_ORANGE).
As you can guess, we have here, timeouts of 5, 5, and 1 seconds:

```C++
	fsm.assignTimeOut( ST_RED,    5, ST_GREEN  );
	fsm.assignTimeOut( ST_GREEN,  5, ST_ORANGE );
	fsm.assignTimeOut( ST_ORANGE, 1, ST_RED   );
```

Once configuration is done, you need to instanciate the timer, assign it to the FSM, and start it:

```C++
	AsioWrapper<STATE,EVENT,std::string> asio;
	fsm.assignTimer( &asio );
	fsm.start();
```

Done !

All things can be found in the runnable example in ```src/traffic_lights_1.cpp```.



TO BE CONTINUED


## Additional facilities

- Printing the configuration:

- Having string identifiers for the events:

- Printing runtime data:


TO BE CONTINUED

## Build options

Several symbols can change the behavior of the library and/or add additional capabilities, you can define them either by adding them in your makefile
(with GCC, its \c -DSPAG_SOME_SYMBOL ), or by hardcoding in your program, BEFORE including the library file, like this:

```C++
#define SPAG_SOME_SYMBOL
#include "spaghetti.hpp"
```

They all start with these 5 characters: ```SPAG_```

The available options/symbols are:
- ```SPAG_PRINT_STATES``` : will print on stdout the steps, useful only for debugging your SpagFSM
- ```SPAG_ENABLE_LOGGING``` : will enable logging of dynamic data (see spag::SpagFSM::printLoggedData() )
- ```SPAG_FRIENDLY_CHECKING```: A lot of checking is done to ensure no nasty bug will crash your program.
However, in case of incorrect usage of the library by your client code (say, invalid index value),
the default behavior is to spit a standard error message that can be difficult to understand.
So if you define this symbol at build time, instead of getting this:
```
myfile: /usr/local/include/spaghetti.hpp:236: void spag::SpagFSM<STATE, EVENT, TIM>::assignTransitionMat(const std::vector<std::vector<T> >&) [with STATE = SERSTAT; EVENT = EN_EVENTS; TIM = AsioWrapper]: Assertion `mat.size() == EVENT::NB_EVENTS' failed.
Aborted
```
you will get this:
```
Spaghetti: runtime error in func: assignTransitionMat(), values are not equal:
 - mat.size() value=7
 - EVENT::NB_EVENTS value=8
Exiting...
```
If this symbol is not defined, regular checking is done with the classical ```assert()```.
As usual, this checking can be removed by defining the symbol ```NDEBUG```.

- ```SPAG_ENUM_STRINGS``` : this enables the usage of enum-string mapping, for events only.
You can provide a string either individually with
```C++
	fsm.assignString2Event( std::make_pair(EV_MyEvent, "something happened" );
```
or globally, by providing a vector of pairs(enum values, string). For example:
```C++
	std::vector<std::pair<EVENT,std::string>> v_str = {
		{ EV_RESET,       "Reset" },
		{ EV_WARNING_ON,  "Warning On" },
		{ EV_WARNING_OFF, "Warning Off" }
	};
	fsm.assignStrings2Events( v_str );
```
These strings will then be printed out when calling the ```printConfig()``` member function.
