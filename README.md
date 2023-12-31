# Characterizing sCMOS sensors
This repository contains code to help characterize and identify noise and other secondary effects that impact sCMOS sensors particuarly for use in astronomy.
  1. [CCD vs CMOS](#ccd-vs-cmos)
  2. [Silicon Defects](#silicon-defects)
  3. [Readout Noise](#readout-noise-ron)
  4. [Dark Current](#dark-current)
  5. [Gain](#gain)
  6. [Linearity](#linearity)
  7. [Charge Persistence](#charge-persistence)
  8. [Salt & Pepper Noise (Random Telegraph Noise)](#salt-and-pepper-noise)

## CCD vs CMOS 
CCD (Charge-Coupled Device) has been the most prevalent imaging sensor type for use in astronomy. In recent years CMOS (Complementary Metal-Oxide-Semiconductor) has dominated the consumer market. As a result CCD sensors are more expensive, harder to find and have less support than their CMOS counterparts. This section details how each sensor works as well as their differences in respect to use in astronomy. 

### CCD Background 
CCD sensors consist of an array of pixels. These pixels are light sensitive elements usually made of silicon. When photons strike these pixels electrons are released. These electrons are transfered through the sensor. Where they are amplified to maintain signal integrety as well as reducing [readout noise](#readout-noise-ron). These electrons are then converted to a descrete signal using a single ADC (analog to digital converter) which converts the electrons coming from each pixel to a digital value. 

### CMOS Background 
CMOS sensors are different in that the instead of a single readout circuit and ADC that reads values from all of the pixels, each pixel has its own readout circitry as well as each row of pixels has its own ADC. This allows for a much faster framerate and more effecient readout process. This is extremely desirable for consumer products like iPhone cameras but for scientific applications poses some issues: 
  1. Readout circuitry on the pixels themselves make it so the sensor has a smaller area exposed to light and a lower sensitivity.
  2. More ADC's provide additional and varied readout noise which cannot be compensated as easily as a CCD sensor.
  3. Noise properties are varied from pixel to pixel which makes characterizing and removing noise a more complex issue. 

![ccd vs cmos diagram](https://www.testandmeasurementtips.com/wp-content/uploads/2019/05/ccd-cmos-image-sensors.jpg) 

# Silicon Defects
Most of the types of noise touched on here are caused due to manufacturing defects when making the semiconductors within a sensor. In an ideal scenario when a certain number of photons hit a sensor, that energy causes electrons within a semiconductor to jump from the valence band to the conduction band. In reality, defects in the silicon cause intermediary bands (traps) in-between the conduction and valence. These bands allow electrons to jump to the conduction band in multiple steps of lower energy. This process is much easier than making it in a single jump. The resulting effect is that the relationship between the drain current and gate voltage in a MOS transistor which discretely increases or decreases the offset This results in some pixels being much more sensitive than others. This effect is also what causes dark current as purely thermal energy can cause the jump between bands.  

 ![image](https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/12a7fa6c-63f1-4767-8a26-4076d25f3c49)

 A more concrete example on how this impacts the operation of FETS can be seen in the drain current equation: 

 
$I_D = \frac{1}{2} \mu_n C_{ox} \frac{W}{L} \left(V_{GS} - V_{th}\right)^2 \$

Where:

$I_D$: Drain current. This is the current flowing from the drain to the source in a MOSFET, representing the flow of charge carriers.

$\mu_n\$: Electron mobility in the channel. It represents the mobility of electrons in the semiconductor material, indicating how easily electrons can move in response to an electric field.

$C_{ox}\$: Oxide capacitance per unit area. It represents the capacitance of the oxide layer in the MOSFET. It is a key parameter influencing the charge storage capacity.

$\frac{W}{L}\$: Width-to-length ratio of the transistor. It reflects the physical dimensions of the MOSFET. The width (\(W\)) and length (\(L\)) of the transistor affect its electrical characteristics.

$\(V_{GS} - V_{th})\$: Gate-source voltage minus the threshold voltage. This term represents the effective voltage applied to the MOSFET to control the channel. When this term is positive, the MOSFET is in the "on" state.

Traps make it harder for electrons to move through the semiconductor as a result the Electron mobility will change which has a direct impact on $I_D$*

* This is not nearly the entire view on how traps impact the operation of FETS. Semiconductor physics is a vast, confusing and hard subject which I am relatively inexperienced in. 

## Readout Noise RON
Located in CMOS CORE/helper directory. 

This is the noise introduced through the readout circitry, mainly the preamplifier and ADC. Usually measured in e- RMS. Follows a skewed histogram distribution as opposed to CCD sensors which a follow a guassian one.

### Procedure 
1. Take 100 bias frames (0s exposure shutter closed) 
2. Perform 3-sigma clipping on each frame, replacing clipped values with the mean of the frame. 
3. Stack them and take pixel wise mean to create a master bias. 
4. Take 545 more bias frames.
5. Subtract the master bias from each of the 545 new bias frames. 
6. For every pixel of coordinate (x,y) calculate: $fract{\sigma}{\sqrt{2}}$ between the N frames.
7. Repeat for every pixel in the frame. This results in an array containing the RON of each pixel 

### Analysis 
For the RON calculation I use the standard deviation over sqrt(2) this is the RMS value not the mean. I chose to do this as the RMS is a better indication of the average due to the skewed histogram distribution. 

1. $\text{RON} = \frac{\sigma}{\sqrt{2}}$

This gives **RON = 1.059 ADU -> 1.27 e-** The histogram is shown below:

![RON-stdOVERsqrt(2)](https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/13eb2e82-4d01-44c8-af9d-a07c169d5a51)
 

$\text{Percent Error} = \frac{1.788-1.9}{1.9} = \text{5.895 Percent}$

The bottleneck of this calucation is the size of memory. During the subtraction the result is a numpy 3D array of size n x 2208 x 3216. With a array of 64 bit floats. The computer I am using has 32 GB RAM -> assuming 31GB usage the most images I can compile is 545. This number will be much lower when running on the pi. This is because when images are loaded their 2d pixel arrays are appended to a list. Optimizations could be made to calculate the RON in batches adding to a running total as they are loaded. 

Running the calculation for **545 Frames at -10.0C** I get the following results:
- Min: 1.194192382537768
- Max: 2.5905072964950713
- **Mean: 1.497172421554032**

![ron-545Frames](https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/74564315-9889-4f66-bb0b-4374178d7279)

In order to fit more images into memory a possible solution is to type the data as a 32 bit float. This would result in loss of precision. 
Running the calculation for the same **545 frames at -10.0C** I get the following results:
- Min: 1.194192886352539
- Max: 2.590508460998535
- **Mean: 1.4971725940704346**

![545-float32RON](https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/1ba7eb21-4778-4ab4-b7e6-8e2814d15e80)
    
## Dark Current
Dark current is the charge buildup on the sensor as a result of heat. This process is light independent noise. For example a specification of 0.5 e-/p/s means every 1 e-/pixel generated per 2 seconds exposure time. This can be reduced through cooling. 
There are three main sources of dark current: 
  1. Diffusion
  2. Thermal generation due to recombination (G-R) of charges
  3. Leakage currents

### Procedure
1. Take N dark frames at multiple different exposure times, same temperature
2. Average N frames and subtract a master bias from them
3. Graph Dark Current Count vs Time. Slope is the dark current

### Analysis 
10 frames were taken at each exposure time: 1s, 10s, 60s, 120s and 240s and the following graph was obtained:

![DCvsTime](https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/274738d7-4186-4fc4-9b9a-0699957306ba)

The resulting DC value is **0.10 e-/p/s**

The dark current is calcualted only for the center 1000x1000 pixel region of the image. The sensor used clearly has areas more succeptable to dark current than others. A exagerated image showing this spots is shown below: 

![image](https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/5886d752-cd93-4dfb-9813-12ff304b60e6)

## Gain 
Gain is the conversion between from arbitrary ADU units to electrons. i.e a gain of 6e-/ADU means there are six electrons per ADU. 

### Procedure 
1. Take N flats at the same exposure time but varying luminance
2. Take N darks at the same exposure time
3. Create Master Dark by taking the mean between N frames
4. For each flat subtract the master dark, then calculate the mean and variance of a nxn area of the image where it is "flattest" (avoid edges due to weird edge effects)
5. Plot Variance vs Mean and calculate the slope, this is the gain. 

### Analysis 
I took a series of flat images all at -5.0C. The exposure time was at 0.2 seconds. My lightsource was my laptop screen with a white image fullscreened. I increased the brightness for each photo and took 6 in total. 

The resulting photon transfer curve (PTC) for the central 300x300 region is shown below: 

![PTC(Good)](https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/cdfe7b4c-a74f-434b-a4b1-3ccf8ad0efbb)

From the inverse slope we can see that the gain is **1.2 e-/ADU**

The RON can also be extracted from the PTC as the $\sqrt{\text{y-intercept}}$ which is **1.6**

I also created a heat map to look at the spacial variance of gain values across the image. 
This works by finding the gain between all n images for a 48x48 pixel block. It then repeates this across the entire image and plots it as a heat map. 

![gainHeatMap](https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/b014805a-1b95-4119-ae74-7069fed701f4)



### Notes
flat frames where pretty good actually (Here is one with a mean luminance ~1000): 
![image](https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/72abdfb9-b631-494c-a5bf-414405d5347c)

## Quantum Efficiency 
Quantum Efficiency (QE) is the measure of how many incident photons are converted into electrons in a sensor. If a sensor was exposed to 100 photons and produced 70 electrons it would have a QE of 70%. 

### Equipment
Light Source - Newport 780 Lamp Supply: 
Just a basic lamp and power source to input light in to the monochromator. The main thing is that it has to monochromatic meaning its relativly stable and outputs all wavelengths of light. 

Monochromator - Oriel 1/8 m Cornerstone Monochromator Model 74000:
This is used to select certain wavelengths of light. It has a diffraction grating inside which moves to output only certain wavelengths. I am only concerned with visible wavelengths. (400-700nm) 

Lens - THORLABS AC254 (D = 25.4mm, F = 45.0mm): 
A simple lens is used. This is to focus light coming out of the monochromator onto the DUT. It should have a coating on it permissible to visible wavelengths 400-700nm

Photodiode - Newport Model 818-SL:
A calibrated photodiode that outputs a power in watts based upon incident light. 

Powermeter - Newport Optical Power Meter Model 1830-C:
Used to read values from the photodiode.  

### Setup 
The lamp is connected to the monochromator. A lens is then placed at the output of the monochromator to focus the output onto the DUT's. The photodiode and sensor are placed on a metal slider so they can easily be moved in and out of the light beam. The whole setup is then covered in dark fabric to reduce any ambient light. 

### Procedure
1. Take exposures at around 80% of saturation across the visible light spectrum: 400nm, 500nm, 600nm and 700nm. 
2. Take darks at same exposure time. 
3. Repeat with photodiode Infront of lens recording the power output from the power meter. 
4. The photodiode term is calculated as:
5. 
Photodiode Term =  \frac{pλ}{hc}

Where: 
p = Power in Watts (From the photodiode)
λ = Wavelength in nm 
h = Planks constant 
c = speed of light 

The sensor term can be calculated as: 

Sensor Term = $\frac{ADU*G}{t}$

Where: 
ADU = Total counts of sensor in ADU
G = gain of sensor
t = exposure time 

The final QE is calculated as: 

QE = $\frac{Sensor Term}{Photodiode Term}$


## Linearity 
Linearity is how sensors behave across different conditions. Particularly how a sensor behaves as it reaches saturation. Does the ratio of incident photons to released electrons remain linear across all saturation levels.

### Procedure
1. Take 30 darks with an exposure time varying from 0s to 120s. 
2. For each dark calculate the mean pixel value in ADU.
3. Graph mean ADU vs Time for all 30 frames.

### Results
The following figure shows the result from the experiment. 
![LinearityTest2_2min_Plot](https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/ef58c934-ea9b-4fd9-960b-32f47b8f8410)

It can be seen that the sensor is indeed linear. Since the sensor is 12 bit saturation occurs at 4096 ADU which is shown with the blue data points. 

## Charge Persistence 
Charge persistence looks at if chargers linger within the sensor from frame to frame. Will a pixel’s value be impacted if it was saturated in the immediately preceding frame.

### Procedure
1. Take a long light exposure so that all pixels are saturated. 
2. Immediately close the shutter then start taking biases in 1s increments.
3. Graph mean ADU vs time for the biases.

### Results
The following chart shows the results from the experiment:

<img width="300" alt="image" src="https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/c292efb1-b82c-4314-9ccd-68460656d999">

It appears that the first image taken after saturation has a slightly higher mean ADU. I would say this is still inconclusive as the variance in mean ADU is extremely small and further investigation should be done. 

## Salt and Pepper Noise
Salt and Pepper Noise also called Random Telegraph Noise (RTN) is discrete but random noise caused by manufacturing defects in silicon which create “traps” it results in more/less sensitive pixels than others.

### Procedure 
1. Take n bias frames. 100 in this case. 
2. Identify a small region of interest (ROI (30x30 pixels) which contains some pixels with a much higher standard deviation.
3. Graph the value of each pixel in ADU for each of the 100 frames.

### Results 
The first result is searching for pixels with a value > mean + 3*std: 

<img alt="image" src="https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/4912e1d6-56ff-4120-bd35-8e54666ac32e">

The analysis was done again this time searching for pixels with a value > mean + 12*std:

<img alt="image" src="https://github.com/aidanmacnichol/CMOS_Characterization/assets/108359181/069aa957-11ab-4cc5-be00-b03fab6fc985">

In both the clustering of average values in the center can be seen. Variation is mainly caused by readout noise here. The outliers are most likely due to S&P noise. In the second image the S&P noise can be seen clustering around 220 ADU.

# Software Setup
First make sure Python is installed. All of the packages can be installed using the command: 

Start up the virtual enviroment from the base directory with: 

'''source venv/bin/activate'''

Then install the required packages with: 

'''pip -r requirments.txt'''

# Hardware Setup
I am using the SBIG STC-428 sCMOS camera from Difraction limited. 

Connect the camera to power, the computer and then plug in the cable from the filter wheel into the "Aux" port. 

Open MaximDL and select the "Toggle Camera Control" icon in the toolbar. Go to the "Setup" tab and under "Camera 1" select "Setup Camera" then in the new window select "Advanced..." select the camera from the Device list on the left should be something like "STC428M-20092901 [USB]" Then hit Ok. 

Then hit the "Setup Filter" option under "Camera 1" select **DL Imaging+FW** from the dropdown list on the right. Then hit Ok and select "Options" on "Camera 1" Make sure the "Use Filter Wheel as Shutter" box is checked. I dont believe the "Filter to use As Shutter" dropdown matters I just have Luminance selected. 

Finally hit "Connect" to set the Cooler temp hit "Cooler" under "Camera 1" 

to take a photo go to the "Expose" tab and hit start when you have selected the desired settings. 


