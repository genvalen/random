# About /dev/random
/dev/random produces random output generated by a CSPRNG (Cryptographically Secure Pseudo Random Number Generator) inside the linux kernel. In the unix-like environment, cryptographically secure random numbers are generated by mixing deterministic, predictable events generated by the computer with events that are random and unpredictable, such as input from a user (keyboard, mouse) or from devices (disk interrupts, timings). Together, these deterministic and random events are thoroughly mixed to create a pool of random bytes that can be used for anything, like encrypted keys. Since the linux kernel has privileged access to information about user, hardware, and device events that can be used as sources of “randomness”, the CSPRNG reads in these events and uses their “randomness” to ensure that it creates an entropy pool whose original values cannot be discovered.

Note: /dev/random is different from /dev/urandom in that it implements blocking; in my understanding, blocking means to halt production of output until the CSPRNG has gathered enough new "randomness" to ensure that the output produced is secure.

# About This Clone
This clone is an approximation of /dev/random. It is limited. It does not interact with the kernel in any way, but it simulates /dev/random in 3 key ways: 
1. Has functionality to collect user input, which in this case, is keyboard input via [sshkeyboard](https://sshkeyboard.readthedocs.io/en/latest/). I chose this library over others for collecting keyboard input because it has no dependencies and doesn't require root access. It uses `fcntl` and `termios` to parse characters from `sys.in`. At this time, the implementation requires the user to do a little bit of manual labor in order to see the output. Given more time, I would have liked to introduce other forms of random user input, such as collecting data from disks, CDs, USBs, or network interfaces. I considered incorporating an OS module to make system calls to interact with the proper linux drivers directly, but this requires further research.
2. Implements a CSPRNG that uses "user input" to add "randomness" to its entropy pool, and produces a stream of random bytes from it. More info in the outline below.
3. Implements blocking: after the CSPRNG produces output, it waits until it has added ~16 more bytes of entropy to the pool before producing output again. In general, the CSPRNG prints random output onto the console for every ~16 bytes of random data it reads in.


### Outline for CSPRNG (randomness algorithm)
1. Initiate an entropy pool as a byte stream of zeros.
2. Wait for a random event/environmental noise.
3. Represent the event numerically, for example, length of time between previous and current instance of the event, the exact time at which event was recorded, etc.
4. Serialize the random event with a low overhead function like sha1 to mimic the CRC hash function used for mixing in /dev/random. 
5. Mix the serialized random event into the entropy pool by concatenating it to the pool, and then creating a new hash, again with sha1, from the bytes in the updated pool.
6. The updated pool becomes the new entropy pool.
7. Repeat from step 2 (Repeat indefinitely, until the program is stopped with `CTRL-C`)
8. When ~16 bytes of entropy have been mixed into the pool, run the blake2s algorithm on the pool (a secure hash) and display the random output in the console. Then, with the mixing function, mix this output back into the entropy pool to update the value of the pool again.


# Requirements
See `requirements.txt` for details.
* Python 3+
* [sshkeyboard](https://sshkeyboard.readthedocs.io/en/latest/)
* [black](https://black.readthedocs.io/en/stable/)


# How to Use
Once you clone the repo and `cd` into the `random` directory, run the program as follows:

```
$ python random.py
```

You will see nothing on the terminal, but the program is running. At this time, press random keys on the keyboard for 2-3 seconds and then press `enter`. 

Repeat this process a few times.

If enough entropy has been gathered while pressing keys, then random output from the CSPRNG's entropy pool will be printed to the console when `enter` is pressed. Otherwise, if no output is printed, this means the program is `blocked` and is waiting on more input. (Press more keys!)

For this project, the amount of new entropy required before generating output from the CSPRNG has been set to 16 bytes so that less manual input is necessary before seeing the output.

Press `CTRL-C` to exit the program.

[Click here to see an output example.](images/screen-shot-2022-01-25.png)

# Limitations
Software 

sshkeyboard does not register key presses very well when the same key is pressed repeatedly in a row, or when multiple keys are pressed at the same time. If you want to see output appear more quickly in the console, it is better to alternate and vary the keys pressed.

Randomness

"Good" randomness requires unpredictability and uniformity. While taking input from the keyboard solves for unpredictability, this input source, and the way I arrange the algorithm, has issues with uniformity. Uniformity is described as ensuring that the source of entropy (keyboard input) allows for equal probability that all bytes, from 0 to 255, are chosen. Currently, my program takes in a char and converts it into its unicode equivalent, an integer also falling between 0 and 255, before moving forward with the remaining steps in the algorithm. Assuming that most often people will click on the lower-case letters `a-z`, and rarely on upper case letters or on other symbols, only a small portion of the unicode range will continually be mixed into the entropy pool. This is a bias that should be addressed to make the CSRNG more unpredictable and secure. Introducing other diverse sources of random entropy could help with this issue.

# Resources
[Entropy in the linux kernel](https://www.youtube.com/watch?v=0DV8WnqhH2Y&t=422s)
* CSPRNG implementation discussed at 7:02-10:40

[random.c file on Github](https://github.com/torvalds/linux/blob/master/drivers/char/random.c)
* Line 54 - general theory.
* Line 122 - kernel's output interfaces.
* Line 193 - functions that collect random events.