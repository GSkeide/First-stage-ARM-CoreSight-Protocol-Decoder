Please read the pdf for a more full depth overview.
<img width="1324" height="496" alt="Screenshot 2026-09-28 194656" src="https://github.com/user-attachments/assets/4ef66516-948f-4256-9c0b-d99d66cfb1f7" />


ARM CoreSight is a huge system of components as shown in this figure:
<img width="1530" height="1294" alt="image" src="https://github.com/user-attachments/assets/52da1be4-7a85-40d1-9bbb-bb11bc16dae4" />

For this project we wanted to extract PTM/ETM instruction trace and trace this data into the FPGA logic, such that we could create a hardware accelerated protocol decoder.
To do this, we chose to utilize the Coresight Access Library (CSAL), which has drivers that deal with the intricate bit flipping of the coresight components and allow us to configure the system to fit our needs.

Our needs are shown below:
<img width="1500" height="1028" alt="image" src="https://github.com/user-attachments/assets/382b7ceb-1901-4add-89e3-6eb3479e4ffa" />

As this was a Collaborative research project, our job was to implement the first stage of this system. 
The overall idea of the complete research project (beyond the scope of our project) is to detect buffer overflows, hacking, etc.. And, if detected, freeze or shut down the system by sending commands through the FTM.


ARM CoreSight has 16 bytes as shown below:
<img width="1552" height="756" alt="image" src="https://github.com/user-attachments/assets/9c9d3050-54c0-447f-a37e-8c5732deddb0" />
To deformat this protocol and extract payloads the entire frame must be assembled, while the TPIU block (which is responsible for routing trace data into the fpga) is not able to send more than 32 bits per clock cycle at most.

Therefor we utilize a "Frame generator" block, who's sole purpose is to "gather" a full frame, this includes filtering out unwanted "synchronization packets" among other things.
A full frame is at most, assembled every 4th clock period. As our project is purposed for security systems we chose to design the system to be able to "keep up" with the maximum possible throughput of 32 bits/clockcycle, thus we have 4 clock cycles to decode a full frame.

It becomes natural to pipeline the logic into a statemachine of 4 states to maximize the clock frequency like this:
<img width="1396" height="1096" alt="image" src="https://github.com/user-attachments/assets/b5347688-2288-4aa5-9b91-c255eb234bda" />

Another requirement was that the decoded data should be stored in memory, for this we chose to utilize AXI DMA, controlled by a AXI_Master with its own internal state machine:
<img width="1652" height="918" alt="image" src="https://github.com/user-attachments/assets/4a96b543-a47f-4d53-be2d-9e9d641b56ac" />

This also allowed us to create a system for automaticly verifying the decoded output by comparing it to the same trace data decoded by an open source arm coresight decoder called "OpenCSD".
<img width="1440" height="1326" alt="image" src="https://github.com/user-attachments/assets/0ddfbc4b-bec4-4b4e-be2c-df01837aaf12" />



In the end, our project was able to meet timing demands on a ZYNQ-7000, which we were given for our part of the project.
<img width="1184" height="308" alt="image" src="https://github.com/user-attachments/assets/e1731049-f33c-4881-abc3-16dd8b73eb28" />
<img width="988" height="278" alt="image" src="https://github.com/user-attachments/assets/078fea73-3646-4a3c-bcdd-a3d5bd67399d" />

The next stages of the collaboritive research project must either update to a Ultrascale+ FPGA if they wish to meet timing demands while allowing the maximum possible throughput, which we see as a necessity for use in embedded security.
Another solution would be to utilize buffers to delay the data and decode at a slower pace than 32bit/cycle. This would most likely be fine in most cases as usually each 32 bit word usually take several hundred clock cycles between, but a potentially malicious actor could stress the CPU to generate abnormal amounts of trace data that could cause buffer overflows with this method.
<img width="1506" height="362" alt="image" src="https://github.com/user-attachments/assets/ac086f9a-a881-4448-a23a-8497e82dcac9" />


<img width="1642" height="478" alt="image" src="https://github.com/user-attachments/assets/f6c90ffb-64d1-4a9d-a7c3-ff49ef00eb85" />

