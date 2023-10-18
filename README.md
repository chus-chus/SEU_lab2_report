# Heart Rate Monitoring with LPC1768
### Jesus Antoñanzas, Nico Ares | SEU, 2023

## Problem Statement

The goal of this project is to implement an algorithm on the LPC1768 microcontroller to process the signal from a commercial heart rate sensor. The algorithm should recognize a signal compatible with a heartbeat and measure its main frequency. The measured heart rate is displayed on a PC console via a serial line, using some hyperterminal software for visualization.

## High level description

At a high level, the algorithm reads data from a heart rate sensor, performs filtering, and uses a sliding window technique to identify peaks corresponding to heartbeats. Finally, the average beats per minute (BPM) is calculated based on the identified peaks and displayed on a PC console via a serial line. 

Here's the conceptual flow:

1.  **Signal Acquisition**: Use an analog pin to read raw heart rate data.
    
2.  **Filtering**: Apply a low-pass filter to smooth the raw signal.
    
3.  **Sliding Window**: Use a buffer of samples to maintain a "window" into the data stream. This window is of a fixed size, and as new data comes in, old data is pushed out.
    
4.  **Peak Detection**: The middle sample in the window is checked against the averages of the samples to the left and right of it. If it's higher than both and exceeds a minimum value, it's considered a peak, indicating a heartbeat.
    
5.  **BPM Calculation**: Keep track of these peaks and their timestamps. Periodically calculate the average time between them to determine the current BPM.
    
6.  **Output**: Display the calculated BPM on a PC console through a serial line.
    

The code is designed to operate within the defined sample frequency.

## Code Structure Overview

```cpp
// Libraries and Constants
// ...

struct heartbeat {
    unsigned short val;
    int timestamp; // ms
};

// Functions
// ...

int main()
{
    // Initialization
    // ...

    while (true) {
        // Signal Processing and Peak Detection
        // ...

        // Sleep and Output
        // ...
    }
}
```

## Detailed Code Logic

### Constants

Constants are defined to control various parameters: sampling frequency (`SAMPLE_FREQ`), length of the vector of peaks (`PEAK_LEN`), rolling window length (`MAX_ROLLING_LEN`), minimum voltage for a sample to be considered a peak (`MIN_V`), length of the lowpass filter's memory (`L`) and minimum distance between peaks (`MIN_BEAT_DISTANCE`).

Note that our project aims to measure heart rates up to a maximum of 150 BPM. Translated into frequency, 150 BPM is 2.5 Hz. Therefore, following the Nyquist Theorem, we would need a minimum sampling rate of 5 Hz, or a maximum time gap of 200 milliseconds between each sample, to accurately capture this rate.

```cpp
#include "mbed.h"
#include <vector>

AnalogIn ain(p15);

#define SAMPLE_FREQ 1
#define PEAK_LEN 10
#define MAX_ROLLING_LEN 11
#define MIN_V 30000
#define L 7
#define MIN_BEAT_DISTANCE 400
```

*Note*: the value of the sample frequency is set as `1` but represents `10` milliseconds. This 10x scaling of time happens across the code and is due to unknown reasons within the embedded system.

### Data Structures

A struct `heartbeat` is used to store each detected heartbeat, which includes the value (`val`) and timestamp (`timestamp`) in milliseconds.

```cpp
struct heartbeat {
    unsigned short val;
    int timestamp;
};
```

### Helper Functions

1. **slide_elements_back()**: Moves the vector elements one index back, effectively shifting the vector to the left.

2. **lowpass_filter()**: A lowpass filter to smooth the incoming sensor data.

### Main Function

### Initialization Block

The main block starts by declaring three vectors: `samples` for storing the filtered sensor values, `raw_samples` for unfiltered data (used only by the lowpass filter), and `peaks` for identified heartbeats. Two timers (`t_iter` and `t_global`) are initialized to keep track of time.

```cpp
std::vector<unsigned short> samples;
std::vector<unsigned short> raw_samples;
std::vector<heartbeat> peaks;
Timer t_iter;
Timer t_global;
```

### The Infinite Loop

This is the core of the program. It's a `while (true)` loop that continually performs sampling, filtering, and peak detection. So, once the system is on, it is continually updating and printing the computed BPM.

#### Sampling and Filtering

The first operation within the loop is to read from the analog pin (`ain.read_u16()`) and pass it through a low-pass filter to smooth out the signal. The filtered value is stored as `new_elem`.

```cpp
unsigned short new_elem = lowpass_filter(ain.read_u16(), raw_samples, accum_smoothing);
```

#### Timer Start

The `t_iter` timer is started to measure the time taken for one iteration of the loop.

```cpp
t_iter.start();
```

#### Rolling Window and Peak Detection

1. **Buffer Filling**: The `samples` vector is filled until it has `MAX_ROLLING_LEN` number of elements.

    ```cpp
    if (samples.size() == MAX_ROLLING_LEN) {
        // Main Logic Here...
    } else {
        samples.push_back(new_elem);
    }
    ```
  
2. **Rolling Mean Calculation**: We calculate rolling means for elements to the left and right of the middle element in the window. The `accum_right` and `accum_left` variables hold the sum of the corresponding sides of the middle element, and `rolling_mean_right` and `rolling_mean_left` store the calculated means.

3. **Peak Detection**: A peak is detected based on the following conditions:

	- **Compare Against Neighbors**: 

	- **Minimum Voltage Threshold**: The middle element should have a value greater than a predefined minimum voltage (`MIN_V`).

	- **Greater Than Neighbors' Mean**: The middle element of the sliding window is compared to the averages of the elements to its left and right. It should be greater than both (`rolling_mean_left` and `rolling_mean_right`).

Specifically, the code checks:
```cpp
if ((rolling_mean_left < samples[middle_elem_pointer]) && (rolling_mean_right < samples[middle_elem_pointer]) 
    && samples[middle_elem_pointer] > MIN_V) {
    // new peak heartbeat
}
```

If all these conditions are met, the sample at the middle of the sliding window is considered to be a peak, indicating a heartbeat.

By setting these conditions, the algorithm ensures that it only counts significant spikes in the data, which likely correspond to actual heartbeats. 

#### Sleeping and Output

To maintain the frequency of the loop (and so the sample frequency), we introduce sleep time, calculated as the difference between `SAMPLE_FREQ` and the elapsed time of the loop iteration (`t_iter.read_ms()`).

```cpp
int sleep_time = SAMPLE_FREQ - t_iter.read_ms();
```

Lastly, the program computes and outputs the heart rate (BPM) approximately every second. BPM are determined by the average time distance between all peaks detected (in the vector of peaks, where old peaks get discarded as new ones are detected).

## Conclusion

The algorithm accomplishes its goal of detecting heart rate in real-time, using a combination of a low-pass filter, a sliding window, and a peak detection algorithm. By adjusting constants like `SAMPLE_FREQ`, `MAX_ROLLING_LEN`, `L`, `MIN_V`, one can fine-tune the algorithm. 

Achieving an optimal setup for these constants has been the most challenging aspect of the project. Balancing sensitivity and accuracy across different individuals has been hard. Our attempts have resulted in an algorithm that is either overly sensitive, triggering false positives, or not very responsive, missing peaks altogether. So, although we've created an adaptable algorithm, achieving consistent performance across a broad spectrum of individuals remains complex. Fine-tuning the system requires that we have an understanding of each parameter's role that the model and that the model developed matches the reality of the sampled values, which has not always been the case.

## Future work

Two key upgrades could make this algorithm better. First, dynamic thresholding would replace the static voltage threshold for smarter peak detection. Second, we could use the mean of multiple central points (instead of comparing just the central point) for more reliable peak identification.
