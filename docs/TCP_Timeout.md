# TCP Timeout Calculation

This formula is used to evaluate the **Final Timeout Value** for TCP retransmission as implemented in Wiznet W5500 firmware:

![image](https://github.com/nmiglio/WiznetW5500/assets/15250689/7e938a50-7f4e-4387-8617-39cd5f4c2e70)

The value is the max time in milliseconds the chip will wait before raising a timeout interrupt.

The variable in the formula are:
- **RTR**: Retry Time-value Register (_default value 2000_)
- **RCR**: Retry Count Register (_default value 8_)
- **M**: Minimum value when RTR × 2<sup>(M+1)</sup> > 65535 and 0 ≤ M ≤ RCR
- **N**: Retransmission count, 0 ≤ N ≤ M
- **RTRMAX**: RTR × 2<sup>M</sup>

As stated in the datasheet for the Retry Time-value Register:
> RTR configures the retransmission timeout period. The unit of timeout period is 100us and the default of RTR is ‘0x07D0’ or ‘2000’.
> And so the default timeout period is 200ms(100us X 2000). During the time configured by RTR, W5500 waits for the peer response to the
> packet that is transmitted by Sn_CR(CONNECT, DISCON, CLOSE, SEND, SEND_MAC, SEND_KEEP command).
> If the peer does not respond within the RTR time, W5500 retransmits the packet or issues timeout.

Basically this sets the amount of time to wait between retransmissions. It is important to notice that in case of TCP timeout this is not a constant value: it will increase at each retransmission.

In the next section of the datasheet for the Retry Count Register:
> RCR configures the number of time of retransmission. When retransmission occurs as many as ‘RCR+1’, Timeout interrupt is issued (Sn_IR[TIMEOUT] = ‘1’).

With this value we sets how many time the chip retry to get a reply. 

Two timeout values are configured with these register: ARP timeout and TCP timeout. 

The ARP timeout is simple:  ![image](https://github.com/nmiglio/WiznetW5500/assets/15250689/45250bfd-2474-48a1-8d58-ad94df9415df)

This is the total time for RCR+1 retransmission (each one takes 𝑅𝑇𝑅 × 0.1𝑚𝑠).

As I anticipated, the TCP timeout is a bit different: it doubles after each retransmission, meaning that if the first transmission has a timeout of 200ms, 
and after this time there is no reply, the second timeout will be 400ms, the third 800ms and so on until the value RTRMAX × 0.1𝑚𝑠 is reached. 
After that if more retransmission are needed the timeout will not be increased anymore.

With RCR = 8 we will have RCR+1=9 retransmission and then the final timeout will be after 31.8 seconds.

Given the complexity of the formula, choosing the right parameters to obtain the desired timeout value is not straight forward.
To have a reference, while working on a project, I derived some values and plotted them to get a nice visual of the results:

![image](https://github.com/nmiglio/WiznetW5500/assets/15250689/c5472c1b-24c9-4fc0-a54a-136459a11c66)

It is to remark that if the RCR register is set to a value lower than M the second part of the equation will not contribute to the final result and the formula should be simplified to:

![image](https://github.com/nmiglio/WiznetW5500/assets/15250689/4ec04997-a2ff-45e9-a479-b52f20de11d7)

It is also interesting to see how choosing RTR as a multiple of 1024 gives the shorter timeout. Choosing a slightly smaller value for the retry tyme (like (1024 × n) - 1) gives a longer total timeout value given the same RCR value. This is because the time between the last two retransmission will still be doubled instead of being RTRMAX = constant (given the way M and RTRMAX are obtained).

This visualization helped me to find the right values for RCR and RTR given the total timeout value I wanted for my system.
When using the default values, the final Timeout is 31.8 seconds (as derived in the W5500 datasheet in the paragraph describing RCR).
In my case I was working with a local network, quite small and without big delays, and I was aiming for a timeout around 6 seconds and at least 4 tries in case of no reply.
Choosing the default value for RTR 2000 and an RCR = 4, gave me a nice 6200ms with RCR + 1 = 5 retransmission.

