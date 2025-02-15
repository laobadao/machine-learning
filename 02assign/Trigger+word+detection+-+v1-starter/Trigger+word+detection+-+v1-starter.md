
# Assignment | 05-week3 -Part_2-Trigger Word Detection

该系列仅在原课程基础上课后作业部分添加个人学习笔记，如有错误，还请批评指教。- ZJ
    
>[Coursera 课程](https://www.coursera.org/specializations/deep-learning) |[deeplearning.ai](https://www.deeplearning.ai/) |[网易云课堂](https://mooc.study.163.com/smartSpec/detail/1001319001.htm)

 [CSDN]()：
   

---

Welcome to the final programming assignment of this specialization! 

In this week's videos, you learned about applying deep learning to speech recognition. In this assignment, you will construct a speech dataset and implement an algorithm for trigger word detection (sometimes also called keyword detection, or wakeword detection). Trigger word detection is the technology that allows devices like Amazon Alexa, Google Home, Apple Siri, and Baidu DuerOS to wake up upon hearing a certain word.  

For this exercise, our trigger word will be "Activate." Every time it hears you say "activate," it will make a "chiming" sound. By the end of this assignment, you will be able to record a clip of yourself talking, and have the algorithm trigger a chime when it detects you saying "activate." 

After completing this assignment, perhaps you can also extend it to run on your laptop so that every time you say "activate" it starts up your favorite app, or turns on a network connected lamp in your house, or triggers some other event? 


欢迎来到这个微专业的最后一篇编程作业！

在本周的视频中，您了解了如何将深度学习应用于语音识别。在这个任务中，您将构建一个语音数据集并实现触发字检测算法（有时也称为关键字检测或唤醒检测）。触发式字词检测技术可让亚马逊 Alexa，Google Home，Apple Siri和百度 DuerOS 等设备在听到某个词语时醒来。

对于这个练习，我们的触发词将是“激活”。每次它听到你说“激活”，它会发出“鸣叫”的声音。在此作业结束时，您将能够录制自己正在讲话的片段，并在算法检测到您说出“激活”时让算法触发一次钟声。

完成这个任务后，也许你还可以扩展它在你的笔记本电脑上运行，这样每当你说“激活”它启动你最喜欢的应用程序，或打开你家的网络连接灯，或触发一些其他事件？

<img src="images/sound.png" style="width:1000px;height:150px;">

In this assignment you will learn to: 
- Structure a speech recognition project 构建语音识别项目
- Synthesize and process audio recordings to create train/dev datasets 合成和处理音频记录以创建 train / dev 数据集
- Train a trigger word detection model and make predictions 训练触发词检测模型并做出预测

Lets get started! Run the following cell to load the package you are going to use.    



```python
import numpy as np
from pydub import AudioSegment
import random
import sys
import io
import os
import glob
import IPython
from td_utils import *
%matplotlib inline
```

    d:\program files\python36\lib\site-packages\pydub\utils.py:165: RuntimeWarning: Couldn't find ffmpeg or avconv - defaulting to ffmpeg, but may not work
      warn("Couldn't find ffmpeg or avconv - defaulting to ffmpeg, but may not work", RuntimeWarning)
    


```python
'''
td_utils.py

'''

import matplotlib.pyplot as plt
from scipy.io import wavfile
import os
from pydub import AudioSegment

# Calculate and plot spectrogram for a wav audio file
def graph_spectrogram(wav_file):
    rate, data = get_wav_info(wav_file)
    nfft = 200 # Length of each window segment
    fs = 8000 # Sampling frequencies
    noverlap = 120 # Overlap between windows
    nchannels = data.ndim
    if nchannels == 1:
        pxx, freqs, bins, im = plt.specgram(data, nfft, fs, noverlap = noverlap)
    elif nchannels == 2:
        pxx, freqs, bins, im = plt.specgram(data[:,0], nfft, fs, noverlap = noverlap)
    return pxx

# Load a wav file
def get_wav_info(wav_file):
    rate, data = wavfile.read(wav_file)
    return rate, data

# Used to standardize volume of audio clip
def match_target_amplitude(sound, target_dBFS):
    change_in_dBFS = target_dBFS - sound.dBFS
    return sound.apply_gain(change_in_dBFS)

# Load raw audio files for speech synthesis
def load_raw_audio():
    activates = []
    backgrounds = []
    negatives = []
    for filename in os.listdir("./raw_data/activates"):
        if filename.endswith("wav"):
            activate = AudioSegment.from_wav("./raw_data/activates/"+filename)
            activates.append(activate)
    for filename in os.listdir("./raw_data/backgrounds"):
        if filename.endswith("wav"):
            background = AudioSegment.from_wav("./raw_data/backgrounds/"+filename)
            backgrounds.append(background)
    for filename in os.listdir("./raw_data/negatives"):
        if filename.endswith("wav"):
            negative = AudioSegment.from_wav("./raw_data/negatives/"+filename)
            negatives.append(negative)
    return activates, negatives, backgrounds
```

# 1 - Data synthesis: Creating a speech dataset 

Let's start by building a dataset for your trigger word detection algorithm. A speech dataset should ideally be as close as possible to the application you will want to run it on. In this case, you'd like to detect the word "activate" in working environments (library, home, offices, open-spaces ...). You thus need to create recordings with a mix of positive words ("activate") and negative words (random words other than activate) on different background sounds. Let's see how you can create such a dataset. 

我们首先为触发字检测算法构建一个数据集。 理想情况下，语音数据集应尽可能接近您希望运行的应用程序。 在这种情况下，您希望在工作环境（图书馆，家庭，办公室，开放空间等）中检测到“激活”一词。 因此，您需要在不同的背景声音中混合使用正面词语（“激活”）和负面词语（除激活以外的随机词）来创建录音。 我们来看看如何创建这样一个数据集。

## 1.1 - Listening to the data   

One of your friends is helping you out on this project, and they've gone to libraries, cafes, restaurants, homes and offices all around the region to record background noises, as well as snippets of audio of people saying positive/negative words. This dataset includes people speaking in a variety of accents. 

In the raw_data directory, you can find a subset of the raw audio files of the positive words, negative words, and background noise. You will use these audio files to synthesize a dataset to train the model. The "activate" directory contains positive examples of people saying the word "activate". The "negatives" directory contains negative examples of people saying random words other than "activate". There is one word per audio recording. The "backgrounds" directory contains 10 second clips of background noise in different environments.

Run the cells below to listen to some examples.

你的一位朋友正在帮助你完成这个项目，并且他们已经去过遍布该地区的图书馆，咖啡馆，餐馆，家庭和办公室，以记录背景噪音，以及人们说正面/负面词汇的片段的片段。 这个数据集包括以各种口音说话的人。

在 raw_data 目录中，您可以找到正面单词，负面单词和背景噪音的原始音频文件的子集。 您将使用这些音频文件合成数据集来训练模型。 “激活”目录包含说“单词激活”的人的正面例子。 “底片”目录包含说除“激活”以外的随机单词的反面例子。 每个音频记录有一个词。 “背景”目录包含10个不同环境下的背景噪音剪辑。

运行下面的单元格来听一些例子。


```python
IPython.display.Audio("./raw_data/activates/1.wav")
```





```python
IPython.display.Audio("./raw_data/negatives/4.wav")
```

              




```python
IPython.display.Audio("./raw_data/backgrounds/2.wav")
# 我本地的 1.wav 0 字节，不可用
```

              



You will use these three type of recordings (positives/negatives/backgrounds) to create a labelled dataset.

## 1.2 - From audio recordings to spectrograms 从录音到声谱图

What really is an audio recording? A microphone records little variations in air pressure over time, and it is these little variations in air pressure that your ear also perceives as sound. You can think of an audio recording is a long list of numbers measuring the little air pressure changes detected by the microphone. We will use audio sampled at 44100 Hz (or 44100 Hertz). This means the microphone gives us 44100 numbers per second. Thus, a 10 second audio clip is represented by 441000 numbers (= $10 \times 44100$). 

It is quite difficult to figure out from this "raw" representation of audio whether the word "activate" was said. In  order to help your sequence model more easily learn to detect triggerwords, we will compute a *spectrogram* of the audio. The spectrogram tells us how much different frequencies are present in an audio clip at a moment in time. 

(If you've ever taken an advanced class on signal processing or on Fourier transforms, a spectrogram is computed by sliding a window over the raw audio signal, and calculates the most active frequencies in each window using a Fourier transform. If you don't understand the previous sentence, don't worry about it.) 

什么是录音？麦克风随着时间的推移记录气压变化很小，正是这些气压的小变化让你的耳朵感觉到了声音。您可以想象一个录音是一个长长的数字列表，用于测量麦克风检测到的微小气压变化。我们将使用以44100赫兹（或44100赫兹）采样的音频。这意味着麦克风每秒给我们44100个数字。因此，一个10秒的音频片段由441000 numbers (= $10 \times 44100$). 表示。

从这个音频的“原始”表示中找出“激活”这个词是否被说出来是相当困难的。为了帮助您的序列模型更容易学习检测触发字，我们将计算音频的*谱图*。频谱图告诉我们一段时间内音频片段中存在多少不同的频率。

（如果你曾经在信号处理或傅里叶变换上采用了先进的课程，通过在原始音频信号上滑动窗口计算频谱图，并使用傅立叶变换计算每个窗口中最活跃的频率，不要理解前面的句子，不要担心。）

Lets see an example. 


```python
IPython.display.Audio("audio_examples/example_train.wav")
```




```python
x = graph_spectrogram("audio_examples/example_train.wav")
```


![png](output_10_0.png)


The graph above represents how active each frequency is (y axis) over a number of time-steps (x axis). 

<img src="images/spectrogram.png" style="width:500px;height:200px;">
<center> **Figure 1**: Spectrogram of an audio recording, where the color shows the degree to which different frequencies are present (loud) in the audio at different points in time. Green squares means a certain frequency is more active or more present in the audio clip (louder); blue squares denote less active frequencies. </center>

音频记录的频谱图，其中颜色显示不同时间点音频中存在（大声）不同频率的程度。 绿色方块意味着音频片段中的某个频率更加活跃或更多（更响亮）; 蓝色方块表示较少的活动频率。

The dimension of the output spectrogram depends upon the hyperparameters of the spectrogram software and the length of the input. In this notebook, we will be working with 10 second audio clips as the "standard length" for our training examples. The number of timesteps of the spectrogram will be 5511. You'll see later that the spectrogram will be the input $x$ into the network, and so $T_x = 5511$.
 
输出谱图的维度取决于谱图软件的超参数和输入的长度。 在本笔记中，我们将使用 10 秒音频剪辑作为我们训练样本的“标准长度”。 频谱图的时间步数将为5511.稍后您会看到频谱图将成为网络中的输入$ x $，and so $T_x = 5511$.



```python
_, data = wavfile.read("audio_examples/example_train.wav")
print("Time steps in audio recording before spectrogram", data[:,0].shape)
print("Time steps in input after spectrogram", x.shape)
```

    Time steps in audio recording before spectrogram (441000,)
    Time steps in input after spectrogram (101, 5511)
    

Now, you can define:


```python
Tx = 5511 # The number of time steps input to the model from the spectrogram 从谱图输入到模型的时间步数
n_freq = 101 # Number of frequencies input to the model at each time step of the spectrogram 在频谱图的每个时间步骤输入到模型的频率数量
```

Note that even with 10 seconds being our default training example length, 10 seconds of time can be discretized to different numbers of value. You've seen 441000 (raw audio) and 5511 (spectrogram). In the former case, each step represents $10/441000 \approx 0.000023$ seconds. In the second case, each step represents $10/5511 \approx 0.0018$ seconds. 

请注意，即使 10 秒作为我们默认的训练样本长度，也可以将10秒的时间离散为不同的数值。 你已经看过 441000（原始音频）和 5511（频谱图）。 在前一种情况下，每一步表示 $10/441000 \approx 0.000023$ 秒。 在第二种情况下，每个步骤代表 $10/5511 \approx 0.0018$ 秒。

For the 10sec of audio, the key values you will see in this assignment are:

- $441000$ (raw audio)
- $5511 = T_x$ (spectrogram output, and dimension of input to the neural network 频谱图输出，以及输入到神经网络的维度). 
- $10000$ (used by the `pydub` module to synthesize audio 由`pydub`模块用于合成音频) 
- $1375 = T_y$ (the number of steps in the output of the GRU you'll build 您将构建的 GRU 输出中的步数). 

Note that each of these representations correspond to exactly 10 seconds of time. It's just that they are discretizing them to different degrees. All of these are hyperparameters and can be changed (except the 441000, which is a function of the microphone). We have chosen values that are within the standard ranges uses for speech systems. 

请注意，这些表示中的每一个对应于恰好10秒的时间。 只是他们正在不同程度地对他们进行离散化。 所有这些都是超参数，可以更改（除了441000，这是麦克风的功能）。 我们选择了语音系统标准范围内的值。

Consider the $T_y = 1375$ number above. This means that for the output of the model, we discretize the 10s into 1375 time-intervals (each one of length $10/1375 \approx 0.0072$s) and try to predict for each of these intervals whether someone recently finished saying "activate." 

Consider also the 10000 number above. This corresponds to discretizing the 10sec clip into 10/10000 = 0.001 second itervals. 0.001 seconds is also called 1 millisecond, or 1ms. So when we say we are discretizing according to 1ms intervals, it means we are using 10,000 steps. 

考虑上面的$ T_y = 1375 $数字。 这意味着，对于模型的输出，我们将 10 秒离散为 1375 个时间间隔（每个长度为$10/1375 \approx 0.0072$s），并尝试预测这些时间间隔是否最近有人说过“激活”。

考虑上面的 10000 个数字。 这对应于将 10 秒剪辑离散化为 10/10000 = 0.001 秒的游程。 0.001 秒也被称为 1 毫秒，或 1 毫秒。 所以当我们说我们按照 1ms 间隔进行离散化时，这意味着我们正在使用 10,000 步。



```python
Ty = 1375 # The number of time steps in the output of our model
```

## 1.3 - Generating a single training example

Because speech data is hard to acquire and label, you will synthesize your training data using the audio clips of activates, negatives, and backgrounds. It is quite slow to record lots of 10 second audio clips with random "activates" in it. Instead, it is easier to record lots of positives and negative words, and record background noise separately (or download background noise from free online sources). 

由于语音数据很难获取和标记，因此您将使用激活，底片和背景的音频剪辑合成训练数据。 记录大量 10 个随机“激活”音频剪辑是很慢的。 相反，记录大量积极和消极词汇，并单独记录背景噪音（或从免费在线渠道下载背景噪音）更容易。

To synthesize a single training example, you will:

- Pick a random 10 second background audio clip
- Randomly insert 0-4 audio clips of "activate" into this 10sec clip
- Randomly insert 0-2 audio clips of negative words into this 10sec clip

Because you had synthesized the word "activate" into the background clip, you know exactly when in the 10sec clip the "activate" makes its appearance. You'll see later that this makes it easier to generate the labels $y^{\langle t \rangle}$ as well. 

因为您已将“激活”一词合成到背景剪辑中，所以您确切知道 10 秒剪辑中何时出现“激活”。 稍后您会看到，这样可以更轻松地生成标签 $y^{\langle t \rangle}$

You will use the pydub package to manipulate audio. Pydub converts raw audio files into lists of Pydub data structures (it is not important to know the details here). Pydub uses 1ms as the discretization interval (1ms is 1 millisecond = 1/1000 seconds) which is why a 10sec clip is always represented using 10,000 steps. 

您将使用 pydub 软件包来处理音频。 Pydub 将原始音频文件转换为 Pydub 数据结构列表（这里了解细节并不重要）。 Pydub 使用 1ms 作为离散化间隔（1ms是1毫秒= 1/1000秒），这就是为什么 10 秒剪辑总是使用 10,000 步表示的原因。


```python
# Load audio segments using pydub 
activates, negatives, backgrounds = load_raw_audio()

print("background len: " + str(len(backgrounds[0])))    # Should be 10,000, since it is a 10 sec clip
print("activate[0] len: " + str(len(activates[0])))     # Maybe around 1000, since an "activate" audio clip is usually around 1 sec (but varies a lot)
print("activate[1] len: " + str(len(activates[1])))     # Different "activate" clips can have different lengths 
```

    background len: 19841
    activate[0] len: 721
    activate[1] len: 731
    

**Overlaying positive/negative words on the background**:

Given a 10sec background clip and a short audio clip (positive or negative word), you need to be able to "add" or "insert" the word's short audio clip onto the background. To ensure audio segments inserted onto the background do not overlap, you will keep track of the times of previously inserted audio clips. You will be inserting multiple clips of positive/negative words onto the background, and you don't want to insert an "activate" or a random word somewhere that overlaps with another clip you had previously added. 

For clarity, when you insert a 1sec "activate" onto a 10sec clip of cafe noise, you end up with a 10sec clip that sounds like someone sayng "activate" in a cafe, with "activate" superimposed on the background cafe noise. You do *not* end up with an 11 sec clip. You'll see later how pydub allows you to do this. 

给定一个 10秒的背景剪辑和一个短的音频剪辑（正面或负面的单词），您需要能够将单词的短片段“添加”或“插入”背景。 为确保插入到背景上的音频片段不重叠，您将跟踪以前插入的音频片段的时间。 您将在背景中插入多个正片/负片单词剪辑，并且您不希望在与您之前添加的另一个剪辑重叠的地方插入“激活”或随机单词。

为了清楚起见，当您在咖啡厅噪音的10秒剪辑中插入1秒“激活”时，最终会出现10秒的剪辑，听起来像某人在咖啡厅中“激活”，并将“激活”叠加在背景咖啡厅噪音上。 你不会以 11 秒的剪辑结束。 稍后你会看到 pydub 如何让你做到这一点。

**Creating the labels at the same time you overlay**:

Recall also that the labels $y^{\langle t \rangle}$ represent whether or not someone has just finished saying "activate." Given a background clip, we can initialize $y^{\langle t \rangle}=0$ for all $t$, since the clip doesn't contain any "activates." 

回想一下，标签 $y^{\langle t \rangle}$代表是否有人刚说完“激活”。 给定一个背景剪辑，我们可以初始化所有$t$的$y^{\langle t \rangle}=0$ ，因为剪辑不包含任何“激活”。

When you insert or overlay an "activate" clip, you will also update labels for $y^{\langle t \rangle}$, so that 50 steps of the output now have target label 1. You will train a GRU to detect when someone has *finished* saying "activate". For example, suppose the synthesized "activate" clip ends at the 5sec mark in the 10sec audio---exactly halfway into the clip. Recall that $T_y = 1375$, so timestep $687 = $ `int(1375*0.5)` corresponds to the moment at 5sec into the audio. So, you will set $y^{\langle 688 \rangle} = 1$. Further, you would quite satisfied if the GRU detects "activate" anywhere within a short time-internal after this moment, so we actually set 50 consecutive values of the label $y^{\langle t \rangle}$ to 1. Specifically, we have $y^{\langle 688 \rangle} = y^{\langle 689 \rangle} = \cdots = y^{\langle 737 \rangle} = 1$.  

当您插入或覆盖“激活”剪辑时，您还将更新$y^{\langle t \rangle}$的标签，以便输出的50个步骤现在具有目标标签 1.您将训练 GRU 以检测何时 有人已*完成*说“激活”。 例如，假设合成的“激活”剪辑在 10 秒音频中的 5 秒标记处结束 - 恰好在剪辑的中途。 回想一下，$T_y = 1375$，所以timetep$687 = $ `int(1375*0.5)`对应于 5 秒进入音频的时刻。 所以，你会设置 $y^{\langle 688 \rangle} = 1$。 此外，如果 GRU 在短时间内在任何时间内检测到“激活” - 在此时刻之后内部，您会非常满意，所以我们实际上将标签 $y^{\langle t \rangle}$ 50 的连续值设置为 1.具体而言， 我们有$y^{\langle 688 \rangle} = y^{\langle 689 \rangle} = \cdots = y^{\langle 737 \rangle} = 1$.  

This is another reason for synthesizing the training data: It's relatively straightforward to generate these labels $y^{\langle t \rangle}$ as described above. In contrast, if you have 10sec of audio recorded on a microphone, it's quite time consuming for a person to listen to it and mark manually exactly when "activate" finished. 

Here's a figure illustrating the labels $y^{\langle t \rangle}$, for a clip which we have inserted "activate", "innocent", activate", "baby." Note that the positive labels "1" are associated only with the positive words. 

这是合成训练数据的另一个原因：如上所述，生成这些标签是相对直接的。 相反，如果您在麦克风上录制了10秒钟的音频，那么听到它并且在“激活”完成时确切地手动标记是相当耗时的。

下面是一张图，说明我们插入"activate", "innocent", activate", "baby." 的剪辑的标签 $y^{\langle t \rangle}$。请注意，正面标签 "1" 只有积极的话语。

<img src="images/label_diagram.png" style="width:500px;height:200px;">
<center> **Figure 2** </center>

To implement the training set synthesis process, you will use the following helper functions. All of these function will use a 1ms discretization interval, so the 10sec of audio is alwsys discretized into 10,000 steps. 

1. `get_random_time_segment(segment_ms)` gets a random time segment in our background audio 在我们的背景音频中获取随机时间段
2. `is_overlapping(segment_time, existing_segments)` checks if a time segment overlaps with existing segments 检查时间段是否与现有段重叠
3. `insert_audio_clip(background, audio_clip, existing_times)` inserts an audio segment at a random time in our background audio using  使用随机时间在我们的背景音频中插入音频段`get_random_time_segment` and `is_overlapping`
4. `insert_ones(y, segment_end_ms)` inserts 1's into our label vector y after the word "activate" 在单词“激活”后插入1到我们的标签向量 y 中，

The function `get_random_time_segment(segment_ms)` returns a random time segment onto which we can insert an audio clip of duration `segment_ms`. Read through the code to make sure you understand what it is doing.  

函数`get_random_time_segment(segment_ms)` 返回一个随机时间段，我们可以在其中插入持续时间段 segment_ms 的音频片段。 仔细阅读代码，以确保您了解它在做什么。



```python
def get_random_time_segment(segment_ms):
    """
    Gets a random time segment of duration segment_ms in a 10,000 ms audio clip.
    
    Arguments:
    segment_ms -- the duration of the audio clip in ms ("ms" stands for "milliseconds")
    
    Returns:
    segment_time -- a tuple of (segment_start, segment_end) in ms
    """
    
    segment_start = np.random.randint(low=0, high=10000-segment_ms)   # Make sure segment doesn't run past the 10sec background 
    segment_end = segment_start + segment_ms - 1
    
    return (segment_start, segment_end)
```

Next, suppose you have inserted audio clips at segments (1000,1800) and (3400,4500). I.e., the first segment starts at step 1000, and ends at step 1800. Now, if we are considering inserting a new audio clip at (3000,3600) does this overlap with one of the previously inserted segments? In this case, (3000,3600) and (3400,4500) overlap, so we should decide against inserting a clip here. 

For the purpose of this function, define (100,200) and (200,250) to be overlapping, since they overlap at timestep 200. However, (100,199) and (200,250) are non-overlapping. 

接下来，假设您在段（1000,1800）和（3400,4500）处插入了音频剪辑。 即，第一段从步骤 1000 开始，并在步骤 1800 结束。现在，如果我们正考虑在（3000,3600）处插入新的音频剪辑，它是否与先前插入的段之一重叠？ 在这种情况下，（3000,3600）和（3400,4500）重叠，所以我们应该决定在这里插入剪辑。

为了这个功能的目的，定义（100,200）和（200,250）是重叠的，因为它们在时间步 200 处重叠。但是，（100,199）和（200,250）是不重叠的。

**Exercise**: Implement `is_overlapping(segment_time, existing_segments)` to check if a new time segment overlaps with any of the previous segments. You will need to carry out 2 steps:

实现 `is_overlapping(segment_time, existing_segments)`以检查新时间段是否与任何之前的段重叠。 您需要执行2个步骤：

1. Create a "False" flag, that you will later set to "True" if you find that there is an overlap. 创建一个“False”标志，如果您发现有重叠，您将稍后设置为“True”。
2. Loop over the previous_segments' start and end times. Compare these times to the segment's start and end times. If there is an overlap, set the flag defined in (1) as True. You can use: 遍历previous_segments的开始和结束时间。 将这些时间与段的开始和结束时间进行比较。 如果有重叠，请将（1）中定义的标志设置为True。 您可以使用：
```python 
for ....:
        if ... <= ... and ... >= ...:
            ...
```
Hint: There is overlap if the segment starts before the previous segment ends, and the segment ends after the previous segment starts.
提示：如果分段在前一个分段结束之前开始，则有重叠，并且该分段在上一个分段开始之后结束。


```python
# GRADED FUNCTION: is_overlapping

def is_overlapping(segment_time, previous_segments):
    """
    Checks if the time of a segment overlaps with the times of existing segments.
    
    Arguments:
    segment_time -- a tuple of (segment_start, segment_end) for the new segment
    previous_segments -- a list of tuples of (segment_start, segment_end) for the existing segments
    
    Returns:
    True if the time segment overlaps with any of the existing segments, False otherwise
    """
    
    segment_start, segment_end = segment_time
    
    ### START CODE HERE ### (≈ 4 line)
    # Step 1: Initialize overlap as a "False" flag. (≈ 1 line)
    overlap = False
    
    # Step 2: loop over the previous_segments start and end times.
    # Compare start/end times and set the flag to True if there is an overlap (≈ 3 lines)
    for previous_start, previous_end in previous_segments:
        if segment_start <= previous_end and segment_end >= previous_start:
            overlap = True
    ### END CODE HERE ###

    return overlap
```


```python
overlap1 = is_overlapping((950, 1430), [(2000, 2550), (260, 949)])
overlap2 = is_overlapping((2305, 2950), [(824, 1532), (1900, 2305), (3424, 3656)])
print("Overlap 1 = ", overlap1)
print("Overlap 2 = ", overlap2)
```

    Overlap 1 =  False
    Overlap 2 =  True
    

**Expected Output**:

<table>
    <tr>
        <td>
            **Overlap 1**
        </td>
        <td>
           False
        </td>
    </tr>
    <tr>
        <td>
            **Overlap 2**
        </td>
        <td>
           True
        </td>
    </tr>
</table>

Now, lets use the previous helper functions to insert a new audio clip onto the 10sec background at a random time, but making sure that any newly inserted segment doesn't overlap with the previous segments. 

**Exercise**: Implement `insert_audio_clip()` to overlay an audio clip onto the background 10sec clip. You will need to carry out 4 steps:

1. Get a random time segment of the right duration in ms.
2. Make sure that the time segment does not overlap with any of the previous time segments. If it is overlapping, then go back to step 1 and pick a new time segment.
3. Add the new time segment to the list of existing time segments, so as to keep track of all the segments you've inserted.  
4. Overlay the audio clip over the background using pydub. We have implemented this for you.

1. 以毫秒为单位获取正确持续时间的随机时间段。
2. 确保时间段与前面的任何时间段都不重叠。 如果它重叠，则返回步骤1并选择新的时间段。
3. 将新时间段添加到现有时间段列表中，以跟踪您插入的所有段。
4. 使用pydub将音频剪辑覆盖在背景上。 我们已经为你实现了这个。


```python
# GRADED FUNCTION: insert_audio_clip

def insert_audio_clip(background, audio_clip, previous_segments):
    """
    Insert a new audio segment over the background noise at a random time step, ensuring that the 
    audio segment does not overlap with existing segments.
    
    Arguments:
    background -- a 10 second background audio recording.  
    audio_clip -- the audio clip to be inserted/overlaid. 
    previous_segments -- times where audio segments have already been placed
    
    Returns:
    new_background -- the updated background audio
    """
    
    # Get the duration of the audio clip in ms
    segment_ms = len(audio_clip)
    
    ### START CODE HERE ### 
    # Step 1: Use one of the helper functions to pick a random time segment onto which to insert 
    # the new audio clip. (≈ 1 line)
    segment_time = get_random_time_segment(segment_ms)
    
    # Step 2: Check if the new segment_time overlaps with one of the previous_segments. If so, keep 
    # picking new segment_time at random until it doesn't overlap. (≈ 2 lines)
    while is_overlapping(segment_time, previous_segments):
        segment_time = get_random_time_segment(segment_ms)

    # Step 3: Add the new segment_time to the list of previous_segments (≈ 1 line)
    previous_segments.append(segment_time)
    ### END CODE HERE ###
    
    # Step 4: Superpose audio segment and background
    new_background = background.overlay(audio_clip, position = segment_time[0])
    
    return new_background, segment_time
```


```python
np.random.seed(5)
audio_clip, segment_time = insert_audio_clip(backgrounds[0], activates[0], [(3790, 4400)])
audio_clip.export("insert_test.wav", format="wav")
print("Segment Time: ", segment_time)
IPython.display.Audio("insert_test.wav")
```

    Segment Time:  (2915, 3635)
    



**Expected Output**

<table>
    <tr>
        <td>
            **Segment Time**
        </td>
        <td>
           (2254, 3169)
        </td>
    </tr>
</table>


```python
# Expected audio
IPython.display.Audio("audio_examples/insert_reference.wav")
```
     



Finally, implement code to update the labels $y^{\langle t \rangle}$, assuming you just inserted an "activate." In the code below, `y` is a `(1,1375)` dimensional vector, since $T_y = 1375$. 

If the "activate" ended at time step $t$, then set $y^{\langle t+1 \rangle} = 1$ as well as for up to 49 additional consecutive values. However, make sure you don't run off the end of the array and try to update `y[0][1375]`, since the valid indices are `y[0][0]` through `y[0][1374]` because $T_y = 1375$. So if "activate" ends at step 1370, you would get only `y[0][1371] = y[0][1372] = y[0][1373] = y[0][1374] = 1`


最后，假设你刚插入一个“激活”，实现代码来更新标签$y^{\langle t \rangle}$。 在下面的代码中，由于$T_y = 1375$，所以`y`是一个`（1,1375）`维向量。

如果“激活”在时间步骤 $t$ 结束，则设置 $y^{\langle t+1 \rangle} = 1$ 以及最多 49 个附加连续值。 但是，请确保您不会跑掉数组的末尾并尝试更新`y [0] [1375]`，因为有效索引是 `y[0][0]`到 `y[0][1374]` 因为 $T_y = 1375$。 因此，如果“激活”在步骤 1370 结束，则将只获得`y[0][1371] = y[0][1372] = y[0][1373] = y[0][1374] = 1`


**Exercise**: Implement `insert_ones()`. You can use a for loop. (If you are an expert in python's slice operations, feel free also to use slicing to vectorize this.) If a segment ends at `segment_end_ms` (using a 10000 step discretization), to convert it to the indexing for the outputs $y$ (using a $1375$ step discretization), we will use this formula:  

实现`insert_ones（）`。 你可以使用for循环。 （如果你是 python 的 slice 操作的专家，那么也可以使用切片来将这个向量化。）如果一个段以`segment_end_ms`结尾（使用10000步离散），将它转换为输出的索引$y$ （使用 $1375$ 步离散化），我们将使用这个公式：

```
    segment_end_y = int(segment_end_ms * Ty / 10000.0)
```


```python
# GRADED FUNCTION: insert_ones

def insert_ones(y, segment_end_ms):
    """
    Update the label vector y. The labels of the 50 output steps strictly after the end of the segment 
    should be set to 1. By strictly we mean that the label of segment_end_y should be 0 while, the
    50 following labels should be ones.
    
    更新标签向量 y。 严格地说，在段结束之后的 50 个输出步骤的标签应该被设置为 1.严格地说，
    我们的意思是 segment_end_y 的标签应该是0，而下面的 50 个标签应该是 1。?????
    
    Arguments:
    y -- numpy array of shape (1, Ty), the labels of the training example
    segment_end_ms -- the end time of the segment in ms
    
    Returns:
    y -- updated labels
    """
    
    # duration of the background (in terms of spectrogram time-steps)
    segment_end_y = int(segment_end_ms * Ty / 10000.0)
    
    # Add 1 to the correct index in the background label (y)
    ### START CODE HERE ### (≈ 3 lines)
    for i in range(segment_end_y+1, segment_end_y+51):
        if i < Ty:
            y[0, i] = 1.0
    ### END CODE HERE ###
    
    return y
```


```python
arr1 = insert_ones(np.zeros((1, Ty)), 9700)
plt.plot(insert_ones(arr1, 4251)[0,:])
print("sanity checks:", arr1[0][1333], arr1[0][634], arr1[0][635])
```

    sanity checks: 0.0 1.0 0.0
    


![png](output_33_1.png)


**Expected Output**
<table>
    <tr>
        <td>
            **sanity checks**:
        </td>
        <td>
           0.0 1.0 0.0
        </td>
    </tr>
</table>
<img src="images/ones_reference.png" style="width:320;height:240px;">

Finally, you can use `insert_audio_clip` and `insert_ones` to create a new training example.

**Exercise**: Implement `create_training_example()`. You will need to carry out the following steps:

1. Initialize the label vector $y$ as a numpy array of zeros and shape $(1, T_y)$.
2. Initialize the set of existing segments to an empty list.
3. Randomly select 0 to 4 "activate" audio clips, and insert them onto the 10sec clip. Also insert labels at the correct position in the label vector $y$.
4. Randomly select 0 to 2 negative audio clips, and insert them into the 10sec clip. 

1. 将标签向量 $y$ 初始化为零和形状 $(1, T_y)$ 的一个 numpy 数组。
2. 将现有段的集合初始化为空列表。
3. 随机选择 0 至 4“激活”音频剪辑，并将其插入 10 秒剪辑。 还要将标签插入标签矢量 $y$ 中的正确位置。
4. 随机选择 0 到 2 个负样本音频片段，并将它们插入 10 秒片段。



```python
# GRADED FUNCTION: create_training_example

def create_training_example(background, activates, negatives):
    """
    Creates a training example with a given background, activates, and negatives.
    
    Arguments:
    background -- a 10 second background audio recording
    activates -- a list of audio segments of the word "activate"
    negatives -- a list of audio segments of random words that are not "activate"
    
    Returns:
    x -- the spectrogram of the training example
    y -- the label at each time step of the spectrogram
    """
    
    # Set the random seed
    np.random.seed(18)
    
    # Make background quieter
    background = background - 20

    ### START CODE HERE ###
    # Step 1: Initialize y (label vector) of zeros (≈ 1 line)
    y = np.zeros((1,Ty))

    # Step 2: Initialize segment times as empty list (≈ 1 line)
    previous_segments = []
    ### END CODE HERE ###
    
    # Select 0-4 random "activate" audio clips from the entire list of "activates" recordings
    number_of_activates = np.random.randint(0, 5)
    random_indices = np.random.randint(len(activates), size=number_of_activates)
    random_activates = [activates[i] for i in random_indices]
    
    ### START CODE HERE ### (≈ 3 lines)
    # Step 3: Loop over randomly selected "activate" clips and insert in background
    for random_activate in random_activates:
        # Insert the audio clip on the background
        background, segment_time = insert_audio_clip(background, random_activate, previous_segments)
        # Retrieve segment_start and segment_end from segment_time
        segment_start, segment_end = segment_time
        # Insert labels in "y"
        y = insert_ones(y, segment_end)
    ### END CODE HERE ###

    # Select 0-2 random negatives audio recordings from the entire list of "negatives" recordings
    number_of_negatives = np.random.randint(0, 3)
    random_indices = np.random.randint(len(negatives), size=number_of_negatives)
    random_negatives = [negatives[i] for i in random_indices]

    ### START CODE HERE ### (≈ 2 lines)
    # Step 4: Loop over randomly selected negative clips and insert in background
    for random_negative in random_negatives:
        # Insert the audio clip on the background 
        background, _ = insert_audio_clip(background, random_negative, previous_segments)
    ### END CODE HERE ###
    
    # Standardize the volume of the audio clip 
    background = match_target_amplitude(background, -20.0)

    # Export new training example 
    file_handle = background.export("train" + ".wav", format="wav")
    print("File (train.wav) was saved in your directory.")
    
    # Get and plot spectrogram of the new recording (background with superposition of positive and negatives)
    x = graph_spectrogram("train.wav")
    
    return x, y
```


```python
x, y = create_training_example(backgrounds[0], activates, negatives)
```

    File (train.wav) was saved in your directory.
    

    d:\program files\python36\lib\site-packages\matplotlib\axes\_axes.py:7172: RuntimeWarning: divide by zero encountered in log10
      Z = 10. * np.log10(spec)
    


![png](output_37_2.png)


**Expected Output**
<img src="images/train_reference.png" style="width:320;height:240px;">

Now you can listen to the training example you created and compare it to the spectrogram generated above.


```python
IPython.display.Audio("train.wav")
```



**Expected Output**


```python
IPython.display.Audio("audio_examples/train_reference.wav")
```



Finally, you can plot the associated labels for the generated training example.


```python
plt.plot(y[0])
```




    [<matplotlib.lines.Line2D at 0x242efc38550>]




![png](output_44_1.png)


**Expected Output**
<img src="images/train_label.png" style="width:320;height:240px;">

## 1.4 - Full training set

You've now implemented the code needed to generate a single training example. We used this process to generate a large training set. To save time, we've already generated a set of training examples. 


```python
# Load preprocessed training examples
X = np.load("./XY_train/X.npy")
Y = np.load("./XY_train/Y.npy")
```

## 1.5 - Development set

To test our model, we recorded a development set of 25 examples. While our training data is synthesized, we want to create a development set using the same distribution as the real inputs. Thus, we recorded 25 10-second audio clips of people saying "activate" and other random words, and labeled them by hand. This follows the principle described in Course 3 that we should create the dev set to be as similar as possible to the test set distribution; that's why our dev set uses real rather than synthesized audio. 

为了测试我们的模型，我们记录了25个例子的开发集。 虽然我们的训练数据是合成的，但我们希望创建一个使用与实际输入相同分布的开发集。 因此，我们记录了人们说“激活”和其他随机单词的 25 个 10 秒钟音频剪辑，并用手标记。 这遵循在课程3中描述的原则，我们应该创建一个尽可能与测试集分布相似的开发集; 这就是为什么我们的开发套件使用真实而不是合成音频。


```python
# Load preprocessed dev set examples
X_dev = np.load("./XY_dev/X_dev.npy")
Y_dev = np.load("./XY_dev/Y_dev.npy")
```

# 2 - Model

Now that you've built a dataset, lets write and train a trigger word detection model! 

The model will use 1-D convolutional layers, GRU layers, and dense layers. Let's load the packages that will allow you to use these layers in Keras. This might take a minute to load. 

现在您已经构建了一个数据集，可以编写和训练一个触发词检测模型！

该模型将使用一维卷积图层，GRU 图层和 dense 图层。 让我们加载可以在 Keras 中使用这些图层的包。 这可能需要一分钟才能加载。


```python
from keras.callbacks import ModelCheckpoint
from keras.models import Model, load_model, Sequential
from keras.layers import Dense, Activation, Dropout, Input, Masking, TimeDistributed, LSTM, Conv1D
from keras.layers import GRU, Bidirectional, BatchNormalization, Reshape
from keras.optimizers import Adam
```

## 2.1 - Build the model

Here is the architecture we will use. Take some time to look over the model and see if it makes sense. 

<img src="images/model.png" style="width:600px;height:600px;">
<center> **Figure 3** </center>

One key step of this model is the 1D convolutional step (near the bottom of Figure 3). It inputs the 5511 step spectrogram, and outputs a 1375 step output, which is then further processed by multiple layers to get the final $T_y = 1375$ step output. This layer plays a role similar to the 2D convolutions you saw in Course 4, of extracting low-level features and then possibly generating an output of a smaller dimension. 

Computationally, the 1-D conv layer also helps speed up the model because now the GRU  has to process only 1375 timesteps rather than 5511 timesteps. The two GRU layers read the sequence of inputs from left to right, then ultimately uses a dense+sigmoid layer to make a prediction for $y^{\langle t \rangle}$. Because $y$ is binary valued (0 or 1), we use a sigmoid output at the last layer to estimate the chance of the output being 1, corresponding to the user having just said "activate."

Note that we use a uni-directional RNN rather than a bi-directional RNN. This is really important for trigger word detection, since we want to be able to detect the trigger word almost immediately after it is said. If we used a bi-directional RNN, we would have to wait for the whole 10sec of audio to be recorded before we could tell if "activate" was said in the first second of the audio clip.  


该模型的一个关键步骤是 1D 卷积步骤（靠近图3的底部）。它输入 5511 步谱图，并输出一个1375步输出，然后再进行多层处理，得到最终的 $T_y = 1375$步输出。该层的作用类似于您在课程 4 中看到的提取低级特征的2D卷积，然后可能会生成较小维度的输出。

从计算角度而言，1-D conv层也有助于加速模型，因为现在 GRU 必须仅处理 1375 个时步而不是 5511 个时步。两个 GRU 层从左到右读取输入序列，然后最终使用  dense+sigmoid 层对$y^{\langle t \rangle}$进行预测。因为 $y$ 是二进制值（0或1），所以我们在最后一层使用 sigmoid 输出来估计输出为1的机会，对应于刚刚说过“激活”的用户。

请注意，我们使用单向RNN而不是双向RNN。这对于触发字检测非常重要，因为我们希望能够在它说出后立即检测触发字。如果我们使用双向RNN，我们必须等待整个10秒的音频被记录下来，然后才能确定音频片段的第一秒是否有“激活”。

Implementing the model can be done in four steps:
    
**Step 1**: CONV layer. Use `Conv1D()` to implement this, with 196 filters, 
a filter size of 15 (`kernel_size=15`), and stride of 4. [[See documentation.](https://keras.io/layers/convolutional/#conv1d)]

**Step 2**: First GRU layer. To generate the GRU layer, use:
```
X = GRU(units = 128, return_sequences = True)(X)
```
Setting `return_sequences=True` ensures that all the GRU's hidden states are fed to the next layer. Remember to follow this with Dropout and BatchNorm layers. 

**Step 3**: Second GRU layer. This is similar to the previous GRU layer (remember to use `return_sequences=True`), but has an extra dropout layer. 

**Step 4**: Create a time-distributed dense layer as follows: 
```
X = TimeDistributed(Dense(1, activation = "sigmoid"))(X)
```
This creates a dense layer followed by a sigmoid, so that the parameters used for the dense layer are the same for every time step. [[See documentation](https://keras.io/layers/wrappers/).]

**Exercise**: Implement `model()`, the architecture is presented in Figure 3.


```python
# GRADED FUNCTION: model

def model(input_shape):
    """
    Function creating the model's graph in Keras.
    
    Argument:
    input_shape -- shape of the model's input data (using Keras conventions)

    Returns:
    model -- Keras model instance
    """
    
    X_input = Input(shape = input_shape)
    
    ### START CODE HERE ###
    
    # Step 1: CONV layer (≈4 lines)
    X = Conv1D(196, 15, strides=4)(X_input)                                 # CONV1D
    X = BatchNormalization()(X)                                 # Batch normalization
    X = Activation('relu')(X)                                 # ReLu activation
    X = Dropout(0.8)(X)                                 # dropout (use 0.8)

    # Step 2: First GRU Layer (≈4 lines)
    X = GRU(units=128, return_sequences=True)(X)                                 # GRU (use 128 units and return the sequences)
    X = Dropout(0.8)(X)                                 # dropout (use 0.8)
    X = BatchNormalization()(X)                                 # Batch normalization
    
    # Step 3: Second GRU Layer (≈4 lines)
    X = GRU(units=128, return_sequences=True)(X)                                 # GRU (use 128 units and return the sequences)
    X = Dropout(0.8)(X)                                 # dropout (use 0.8)
    X = BatchNormalization()(X)                                 # Batch normalization
    X = Dropout(0.8)(X)                              # dropout (use 0.8)
    
    # Step 4: Time-distributed dense layer (≈1 line)
    X = TimeDistributed(Dense(1, activation = "sigmoid"))(X) # time distributed  (sigmoid)

    ### END CODE HERE ###

    model = Model(inputs = X_input, outputs = X)
    
    return model  
```


```python
model = model(input_shape = (Tx, n_freq))
```

Let's print the model summary to keep track of the shapes.


```python
model.summary()
```

    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    input_2 (InputLayer)         (None, 5511, 101)         0         
    _________________________________________________________________
    conv1d_2 (Conv1D)            (None, 1375, 196)         297136    
    _________________________________________________________________
    batch_normalization_3 (Batch (None, 1375, 196)         784       
    _________________________________________________________________
    activation_2 (Activation)    (None, 1375, 196)         0         
    _________________________________________________________________
    dropout_4 (Dropout)          (None, 1375, 196)         0         
    _________________________________________________________________
    gru_3 (GRU)                  (None, 1375, 128)         124800    
    _________________________________________________________________
    dropout_5 (Dropout)          (None, 1375, 128)         0         
    _________________________________________________________________
    batch_normalization_4 (Batch (None, 1375, 128)         512       
    _________________________________________________________________
    gru_4 (GRU)                  (None, 1375, 128)         98688     
    _________________________________________________________________
    dropout_6 (Dropout)          (None, 1375, 128)         0         
    _________________________________________________________________
    batch_normalization_5 (Batch (None, 1375, 128)         512       
    _________________________________________________________________
    dropout_7 (Dropout)          (None, 1375, 128)         0         
    _________________________________________________________________
    time_distributed_1 (TimeDist (None, 1375, 1)           129       
    =================================================================
    Total params: 522,561
    Trainable params: 521,657
    Non-trainable params: 904
    _________________________________________________________________
    

**Expected Output**:

<table>
    <tr>
        <td>
            **Total params**
        </td>
        <td>
           522,561
        </td>
    </tr>
    <tr>
        <td>
            **Trainable params**
        </td>
        <td>
           521,657
        </td>
    </tr>
    <tr>
        <td>
            **Non-trainable params**
        </td>
        <td>
           904
        </td>
    </tr>
</table>

The output of the network is of shape (None, 1375, 1) while the input is (None, 5511, 101). The Conv1D has reduced the number of steps from 5511 at spectrogram to 1375. 

网络的输出是形状（None，1375,1），而输入是（None，5511,101）。 Conv1D 将光谱图中的 5511 步数减少到 1375。

## 2.2 - Fit the model

Trigger word detection takes a long time to train. To save time, we've already trained a model for about 3 hours on a GPU using the architecture you built above, and a large training set of about 4000 examples. Let's load the model. 

触发词检测需要很长时间来训练。 为了节省时间，我们已经使用上面构建的体系结构在GPU上训练了约3小时的模型，并为大约4000个示例进行了大量训练。 让我们加载模型。


```python
model = load_model('./models/tr_model.h5')
```

You can train the model further, using the Adam optimizer and binary cross entropy loss, as follows. This will run quickly because we are training just for one epoch and with a small training set of 26 examples. 

您可以使用 Adam优化 和二元交叉熵损失进一步训练模型，如下所示。 这将会很快发生，因为我们正在为一个epoch 进行训练，并且只有 26 个样本的训练集。



```python
opt = Adam(lr=0.0001, beta_1=0.9, beta_2=0.999, decay=0.01)
model.compile(loss='binary_crossentropy', optimizer=opt, metrics=["accuracy"])
```


```python
model.fit(X, Y, batch_size = 5, epochs=1)
```

    Epoch 1/1
    26/26 [==============================] - 11s 427ms/step - loss: 0.0558 - acc: 0.9797
    




    <keras.callbacks.History at 0x242938d8da0>



## 2.3 - Test the model

Finally, let's see how your model performs on the dev set.


```python
loss, acc = model.evaluate(X_dev, Y_dev)
print("Dev set accuracy = ", acc)
```

    25/25 [==============================] - 2s 68ms/step
    Dev set accuracy =  0.9355927109718323
    

This looks pretty good! However, accuracy isn't a great metric for this task, since the labels are heavily skewed to 0's, so a neural network that just outputs 0's would get slightly over 90% accuracy. We could define more useful metrics such as F1 score or Precision/Recall. But let's not bother with that here, and instead just empirically see how the model does. 

这看起来不错！ 然而，对于这项任务来说，准确性并不是一个很好的指标，因为标签严重倾斜到 0，所以只输出 0 的神经网络的准确性会略高于90％。 我们可以定义更多有用的指标，如 F1 分数或 Precision / Recall。 但是，让我们不要在这里烦恼，而只是凭经验去看模型是如何工作的。

# 3 - Making Predictions

Now that you have built a working model for trigger word detection, let's use it to make predictions. This code snippet runs audio (saved in a wav file) through the network. 

<!--
can use your model to make predictions on new audio clips.

You will first need to compute the predictions for an input audio clip.

**Exercise**: Implement predict_activates(). You will need to do the following:

1. Compute the spectrogram for the audio file
2. Use `np.swap` and `np.expand_dims` to reshape your input to size (1, Tx, n_freqs)
5. Use forward propagation on your model to compute the prediction at each output step
!-->


```python
def detect_triggerword(filename):
    plt.subplot(2, 1, 1)

    x = graph_spectrogram(filename)
    # the spectogram outputs (freqs, Tx) and we want (Tx, freqs) to input into the model
    x  = x.swapaxes(0,1)
    x = np.expand_dims(x, axis=0)
    predictions = model.predict(x)
    
    plt.subplot(2, 1, 2)
    plt.plot(predictions[0,:,0])
    plt.ylabel('probability')
    plt.show()
    return predictions
```

Once you've estimated the probability of having detected the word "activate" at each output step, you can trigger a "chiming" sound to play when the probability is above a certain threshold. Further, $y^{\langle t \rangle}$ might be near 1 for many values in a row after "activate" is said, yet we want to chime only once. So we will insert a chime sound at most once every 75 output steps. This will help prevent us from inserting two chimes for a single instance of "activate". (This plays a role similar to non-max suppression from computer vision.) 


一旦你估计了在每个输出步骤检测到“激活”这个词的概率，当概率超过某个阈值时，你就可以触发“鸣叫”声音。 此外，在“activate”之后， $y^{\langle t \rangle}$对于连续的许多值可能接近1，但我们只想要一次钟声。 所以我们每75个输出步骤最多插入一次钟声。 这将有助于防止我们为单个“激活”实例插入两个钟声。 （这起到类似于计算机视觉非最大抑制的作用。）

<!-- 
**Exercise**: Implement chime_on_activate(). You will need to do the following:

1. Loop over the predicted probabilities at each output step
2. When the prediction is larger than the threshold and more than 75 consecutive time steps have passed, insert a "chime" sound onto the original audio clip

Use this code to convert from the 1,375 step discretization to the 10,000 step discretization and insert a "chime" using pydub:

` audio_clip = audio_clip.overlay(chime, position = ((i / Ty) * audio.duration_seconds)*1000)
`
!--> 


```python
chime_file = "audio_examples/chime.wav"
def chime_on_activate(filename, predictions, threshold):
    audio_clip = AudioSegment.from_wav(filename)
    chime = AudioSegment.from_wav(chime_file)
    Ty = predictions.shape[1]
    # Step 1: Initialize the number of consecutive output steps to 0
    consecutive_timesteps = 0
    # Step 2: Loop over the output steps in the y
    for i in range(Ty):
        # Step 3: Increment consecutive output steps
        consecutive_timesteps += 1
        # Step 4: If prediction is higher than the threshold and more than 75 consecutive output steps have passed
        if predictions[0,i,0] > threshold and consecutive_timesteps > 75:
            # Step 5: Superpose audio and background using pydub
            audio_clip = audio_clip.overlay(chime, position = ((i / Ty) * audio_clip.duration_seconds)*1000)
            # Step 6: Reset consecutive output steps to 0
            consecutive_timesteps = 0
        
    audio_clip.export("chime_output.wav", format='wav')
```

## 3.3 - Test on dev examples

Let's explore how our model performs on two unseen audio clips from the development set. Lets first listen to the two dev set clips. 


```python
IPython.display.Audio("./raw_data/dev/1.wav")
```





```python
IPython.display.Audio("./raw_data/dev/2.wav")
```



Now lets run the model on these audio clips and see if it adds a chime after "activate"! 
现在让我们在这些音频剪辑上运行模型，看看它是否在“激活”之后添加了铃声！


```python
filename = "./raw_data/dev/1.wav"
prediction = detect_triggerword(filename)
chime_on_activate(filename, prediction, 0.5)
IPython.display.Audio("./chime_output.wav")
```


![png](output_78_0.png)




```python
filename  = "./raw_data/dev/2.wav"
prediction = detect_triggerword(filename)
chime_on_activate(filename, prediction, 0.5)
IPython.display.Audio("./chime_output.wav")
```


![png](output_79_0.png)




# Congratulations 

You've come to the end of this assignment! 

Here's what you should remember:
- Data synthesis is an effective way to create a large training set for speech problems, specifically trigger word detection. 
- Using a spectrogram and optionally a 1D conv layer is a common pre-processing step prior to passing audio data to an RNN, GRU or LSTM.
- An end-to-end deep learning approach can be used to built a very effective trigger word detection system. 

- 数据合成是为语音问题创建大型训练集的有效方法，特别是触发词检测。
- 在将音频数据传递到RNN，GRU或LSTM之前，使用频谱图和可选的1D conv层是常见的预处理步骤。
- 端到端的深度学习方法可用于构建非常有效的触发字检测系统。

*Congratulations* on finishing the fimal assignment! 

Thank you for sticking with us through the end and for all the hard work you've put into learning deep learning. We hope you have enjoyed the course! 


# 4 - Try your own example! (OPTIONAL/UNGRADED)

In this optional and ungraded portion of this notebook, you can try your model on your own audio clips! 

Record a 10 second audio clip of you saying the word "activate" and other random words, and upload it to the Coursera hub as `myaudio.wav`. Be sure to upload the audio as a wav file. If your audio is recorded in a different format (such as mp3) there is free software that you can find online for converting it to wav. If your audio recording is not 10 seconds, the code below will either trim or pad it as needed to make it 10 seconds. 


在这个笔记的可选和未定义部分中，您可以在自己的音频剪辑上尝试模型！

录制10秒钟的音频片段，说出“激活”和其他随机单词，然后将其作为myaudio.wav上传到Coursera中心。 确保将音频上传为wav文件。 如果您的音频以不同的格式（如mp3）录制，则可以在线找到用于将其转换为wav的免费软件。 如果您的录音时间不是10秒钟，下面的代码会根据需要进行修剪或填充，以使其达到10秒。


```python
# Preprocess the audio to the correct format
def preprocess_audio(filename):
    # Trim or pad audio segment to 10000ms
    padding = AudioSegment.silent(duration=10000)
    segment = AudioSegment.from_wav(filename)[:10000]
    segment = padding.overlay(segment)
    # Set frame rate to 44100
    segment = segment.set_frame_rate(44100)
    # Export as wav
    segment.export(filename, format='wav')
```

Once you've uploaded your audio file to Coursera, put the path to your file in the variable below.


```python
your_filename = "audio_examples/my_audio.wav"
```


```python
preprocess_audio(your_filename)
IPython.display.Audio(your_filename) # listen to the audio you uploaded 
```


            
Finally, use the model to predict when you say activate in the 10 second audio clip, and trigger a chime. If beeps are not being added appropriately, try to adjust the chime_threshold.


```python
chime_threshold = 0.5
prediction = detect_triggerword(your_filename)
chime_on_activate(your_filename, prediction, chime_threshold)
IPython.display.Audio("./chime_output.wav")
```


![png](output_87_0.png)



              