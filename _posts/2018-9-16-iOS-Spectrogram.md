---
layout: post
title: How build a naive spectrogram on iOS?
---

![Alt Image Text]({{ site.baseurl }}/images/FFT_1/SpectrographTop.jpg "Spectrogam")
  
## First of all, what is a spectrogram?

It is basically a graphical representation of the magnitudes of a set of frequencies at a given time.

- It means that to build an image that looks like the one on the top of this article, we need:
	* A set of frequencies
	* Their respective magnitudes for a given time.

![Alt Image Text]({{ site.baseurl }}/images/FFT_1/AP_Graphic_Elements_I.jpg "Spectrogam")

To get that kind of information we can use a simple tool called FFT.  
FFT stands for Fast Fourier Transform.  
Complexity explained
[here](http://mathworld.wolfram.com/FastFourierTransform.html) and [here](http://www.dspguide.com/ch12/2.htm)

> ### _Intuition_ [^Point]
* This tool is used to go from continuous world to discrete world.
* This tool is used to go from digital world to numerical world.
* This tool is used to go from real world to computer world.  


So we can use it to get a computer compatible representation of a real world signal, such as for example: sound.

![Alt Image Text]({{ site.baseurl }}/images/FFT_1/AP_Graphic_Elements_II.jpg "Spectrogam")

Finally we'll try to build a two step tool:

![Alt Image Text]({{ site.baseurl }}/images/FFT_1/AP_Graphic_Elements_III.jpg "Spectrogam")

By chance, performing the FFT of a signal (basically an array of FloatingPoints values) is a very common task,
so we can use existing library.

Apple has built Accelerate(link) which is a powerfull and optimized (for iOS devices) BLAS (Basic Linear Algebra Subprograms) (link).  
For a more "in-depth" travel in Apple Accelerate framework usage for FFT see this article(link).

To be sure that we use the Accelerate framework appropriatly, we need some test datas.
Here are some nice explanation: http://www.sccon.ca/sccon/fft/fft3.htm

In short the result of an FFT is Imaginary and the other, real.
The real part is the **magnitudes** for a given **frequency**, the Imaginary part will be used later.
   
Go to the code!

### Perform the FFT 

```
public func fft(_ input: [Float]) -> (real:[Float], img:[Float]) {
    
    // Input is the real part
    var real = input
    let size = real.count
    
    // prepare a recipient for the Imaginary part (filled with zero by default)
    var imaginary = [Float](repeating: 0.0, count: size)
    var splitComplex = DSPSplitComplex(realp: &real, imagp: &imaginary)
    
    // the size has to be a power of 2
    let length = vDSP_Length(floor(log2(Float(size))))
    let radix = FFTRadix(kFFTRadix2)
    let weights = vDSP_create_fftsetup(length, radix)
    
    // perform the FFT
    vDSP_fft_zip(weights!, &splitComplex, 1, length, FFTDirection(FFT_FORWARD))
    
    // clean
    vDSP_destroy_fftsetup(weights)
    
    return (real,imaginary)
}
```

### Test in on Playground

_Using a sine wave as input for the fft function:_

![Alt Image Text]({{ site.baseurl }}/images/FFT_1/sin_fft.png "Sin FFT")
 
> As we can see, the real part is _mirrored_ [^MirroredFFT].  
So in our case, we will use only the first half of this result.  


## First steps to the Spectrogram

Three informations are available in the spectrogram built here:

- [X] Magnitudes
- [ ] Time
- [ ] Frequencies


### How to know at what time this magnitude appears?

Let's use a sound file to understand how it works.  
> A sound file has a sample rate, for example: 8kHz.  
It means, that it store 8k informations for 1 second.

On the other side, for a given array of samples, and a known sample rate, we can get the "duration" of this array.

Basically:  

```

numberOfSample/sampleRate = totalDuration

```

To get more precise information on a small part of that audio file, we need to chunk it and give that chunk to the fft function.  

> Generally: the small part given to the fft function is a power of 2.

Let say: _for a chunks of 1024 for a sample rate of 8kHz, what kind of duration is it?_

```

1024/8000 = 0.128s

```

The time interval in this case is: 0.128s.  
In this context 1024 samples represent 0.128s of the recorded sound 

![Alt Image Text]({{ site.baseurl }}/images/FFT_1/AP_Graphic_Elements_IV.jpg "Spectrogam")

- [X] Magnitudes
- [X] Time
- [ ] Frequencies



### How to correlate frequencies with the computed magnitudes?

There is a simple rule based on the chunk given to the FFT and the sample rate:

```

sampleRate/chunkSize = frequencyStep

```

To be more concrete: ```44100Hz/1024 = 43.07Hz```

It means that the correlated frequecy of an index of our fft result is ```frequencyStep*index```

Code, code, code:

##### Retrive frequency step

```
public func frequencyStepForIndex(_ index:Float) -> Float {
    return index*sampleRate / fftSize
}
```

As we can see, the bigger is the chunck size, the smaller is the frequency step.  
But also, the smaller is the sample rate, the smaller needs to be the chunk size!

- [X] Magnitudes
- [X] Time
- [X] Frequencies

DONE!

Lets recap:

1) Extract sample from audio sound file:

```
func loadArrayOf<T:FloatingPoint>(_ type:T.Type, url : URL, sampleRate:Double = 44100) -> [T]? {
     var samples = [T]()
     do {
        let data = try Data(contentsOf: url)
            
        let file = try! AVAudioFile(forReading: url)
        let format = AVAudioFormat(commonFormat: .pcmFormatFloat32, sampleRate: Double(sampleRate), channels: 1, interleaved: true)
            
        if let buffer = AVAudioPCMBuffer(pcmFormat: format!, frameCapacity: AVAudioFrameCount(data.count)){
            try! file.read(into: buffer)
                
            let arraySize = Int(buffer.frameLength)
                
            switch type {
            case is Double.Type:
                var doublePointer = UnsafeMutablePointer<Double>.allocate(capacity: arraySize)
                vDSP_vspdp(buffer.floatChannelData![0], 1, doublePointer, 1, vDSP_Length(arraySize))
                return Array(UnsafeBufferPointer(start: doublePointer, count:arraySize)) as? [T]
            case is Float.Type:
                return Array(UnsafeBufferPointer(start: buffer.floatChannelData![0], count:arraySize)) as? [T]
            default: return nil
            }
            
        }
    } catch let error as NSError {
        print("ERROR HERE", error.localizedDescription)
    }
    return nil
}
```

2) Extract chunks of a given size _(needs to be a power of two)_:
##### Easier with that extension, right?

```
public extension Array {
    func chunks(_ chunkSize: Int) -> [[Element]] {
        return stride(from: 0, to: self.count, by: chunkSize).map {
            Array(self[$0..<Swift.min($0 + chunkSize, self.count)])
        }
    }
}

```

3) Give the chunks to the fft function and store the results

![Alt Image Text]({{ site.baseurl }}/images/FFT_1/rawAudio+FFT.png "Audio FFT")


_Not a really usefull data representation..._
  

### Lets build the image

A common spectrogram is visually built this way:

| Fn   	|  MagT0_n  	|  MagT1_n  	|     	|     	|     	|     	|     	|     	|  MagTn_n  	|
|------	|:---------:	|:---------:	|:---:	|:---:	|:---:	|:---:	|:---:	|:---:	|:---------:	|
| F151 	| MagT0_151 	| MagT1_151 	| ... 	| ... 	| ... 	| ... 	|     	|     	| MagTn_151 	|
| F150 	| MagT0_150 	| MagT1_150 	| ... 	| ... 	| ... 	|     	|     	|     	| MagTn_150 	|
| ...  	|    ...    	|    ...    	| ... 	| ... 	| ... 	|     	|     	|     	|           	|
| F3   	|  MagT0_3  	|  MagT1_3  	| ... 	| ... 	| ... 	| ... 	| ... 	|     	|  MagTn_3  	|
| F2   	|  MagT0_2  	|  MagT1_2  	| ... 	| ... 	|     	|     	|     	|     	|  MagTn_2  	|
| F1   	|  MagT0_1  	|  MagT1_1  	| ... 	| ... 	|     	|     	|     	|     	|  MagTn_1  	|
| F0   	|  MagT0_0  	|  MagT1_0  	| ... 	| ... 	|     	|     	|     	|     	|  MagTn_0  	|
|      	|     T0    	|     T1    	|  T2 	|  T3 	|  T4 	|  T5 	|  T6 	| ... 	|     Tn    	|



  
  
Due to the powerfull but C fashioned CoreGraphics library, It's not that simple to play with pixels value on iOS.  
Thanksfully, here is a nice library named [EasyImagy](https://github.com/koher/EasyImagy)

#### Building image from magnitudes 

```
static func drawMagnitudes(magnitudes:[[Float]], width:Int? = nil, height:Int? = nil) -> UIImage? {
        
    let timeWidth = magnitudes.count
    let frequencyHeight = magnitudes.first!.count
        
    let pixels = magnitudes.map{ $0.map{ RGBA<Float>(gray: 1-$0) } }
        
    let flattenedPixels = pixels.flatMap{ $0 }
        
    var image = Image<RGBA<Float>>(width: frequencyHeight, height:timeWidth , pixels: flattenedPixels)
    image = image.rotated(byDegrees: -90)
        
    if let w = width,
        let h = height {
        image = image.resizedTo(width: w, height: h)
    }
        
    return image.uiImage
}
```

#### Try it 
![Alt Image Text]({{ site.baseurl }}/images/FFT_1/spectroFirst.png "Audio FFT")

> ### Evaluation:
We see a symetry on this picture.

_We need to keep **half** of the FFT results_

#### Try it 
![Alt Image Text]({{ site.baseurl }}/images/FFT_1/spectroSecond.png "Audio FFT")

--- 

> ### Evaluation:
We can compare our _toy spectrogram builder_ with a spectrogram build from the well known library: [SOX](http://sox.sourceforge.net)


| SOX        | TOY           |
| ------------- |:-------------:|
|   ![Alt Image Text]({{ site.baseurl }}/images/FFT_1/bonjourSpectro.png "Audio FFT")    | ![Alt Image Text]({{ site.baseurl }}/images/FFT_1/toySpectro.png "Audio FFT") |

As we can see, the shape of the displayed magnitudes is roughly the same. But its like if our toy spectrogram has a smaller definition.  
We'll take care of that point on the next article!

Still a long way to get a real spectrogram, but enought for today!

---

Here is the full project: https://github.com/AlbanPerli/iOS-Spectrogram

---



[^Point]: just trying to use basic sentences to get different point of views, isn't it the starting point for intuition?

[^MirroredFFT]: https://dsp.stackexchange.com/questions/4825/why-is-the-fft-mirrored/4827#4827
