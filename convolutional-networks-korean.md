---
layout: page
permalink: /convolutional-networks/
---

Table of Contents:

- [Architecture Overview](#overview)
- [ConvNet Layers](#layers)
  - [Convolutional Layer](#conv)
  - [Pooling Layer](#pool)
  - [Normalization Layer](#norm)
  - [Fully-Connected Layer](#fc)
  - [Converting Fully-Connected Layers to Convolutional Layers](#convert)
- [ConvNet Architectures](#architectures)
  - [Layer Patterns](#layerpat)
  - [Layer Sizing Patterns](#layersizepat)
  - [Case Studies](#case) (LeNet / AlexNet / ZFNet / GoogLeNet / VGGNet)
  - [Computational Considerations](#comp)
- [Additional References](#add)

## 컨볼루션 신경망 (CNN/ConvNets)

컨볼루션 신경망 (Convolutional Neural Network, 이하 CNN)은 앞 장에서 다룬 일반 신경망과 매우 유사하다. CNN은 학습 가능한 가중치 (weight)와 바이어스(bias)로 구성되어 있다. 각 뉴런은 입력을 받아 내적 연산( dot product )을 한 뒤 선택에 따라 비선형 (non-linear) 연산을 한다. 전체 네트워크는 일반 신경망과 마찬가지로 미분 가능한 하나의 스코어 함수 (score function)을 갖게 된다 (맨 앞쪽에서 로우 이미지 (raw image)를 읽고 맨 뒤쪽에서 각 클래스에 대한 점수를 구하게 됨). 또한 CNN은 마지막 레이어에 (SVM/Softmax와 같은) 손실 함수 (loss function)을 가지며, 우리가 일반 신경망을 학습시킬 때 사용하던 각종 기법들을 동일하게 적용할 수 있다.

CNN과 일반 신경망의 차이점은 무엇일까? CNN 아키텍쳐는 입력 데이터가 이미지라는 가정 덕분에 이미지 데이터가 갖는 특성들을 인코딩 할 수 있다. 이러한 아키텍쳐는 포워드 함수 (forward function)을 더욱 효과적으로 구현할 수 있고 네트워크를 학습시키는데 필요한 모수 (parameter)의 수를 크게 줄일 수 있게 해준다.

<a name='overview'></a>

### 아키텍쳐 개요

앞 장에서 보았듯이 신경망은 입력받은 벡터를 일련의 히든 레이어 (hidden layer) 를 통해 변형 (transform) 시킨다. 각 히든 레이어는 뉴런들로 이뤄져 있으며, 각 뉴런은 앞쪽 레이어 (previous layer)의 모든 뉴런과 연결되어 있다 (fully connected). 같은 레이어 내에 있는 뉴런들 끼리는 연결이 존재하지 않고 서로 독립적이다. 마지막 Fully-connected 레이어는 출력 레이어라고 불리며, 분류 문제에서 클래스 점수 (class score)를 나타낸다.

일반 신경망은 이미지를 다루기에 적절하지 않다. CIFAR-10 데이터의 경우 각 이미지가 32x32x3 (가로,세로 32, 3개 컬러 채널)로 이뤄져 있어서 첫 번째 히든 레이어 내의 하나의 뉴런의 경우 32x32x3=3072개의 가중치가 필요하지만, 더 큰 이미지를 사용할 경우에는 같은 구조를 이용하는 것이 불가능하다. 예를 들어 200x200x3의 크기를 가진 이미지는 같은 뉴런에 대해 200x200x3=120,000개의 가중치를 필요로 하기 때문이다. 더욱이, 이런 뉴런이 레이어 내에 여러개 존재하므로 모수의 개수가 크게 증가하게 된다. 이와 같이 Fully-connectivity는 심한 낭비이며 많은 수의 모수는 곧 오버피팅(overfitting)으로 귀결된다. 

CNN은 입력이 이미지로 이뤄져 있다는 특징을 살려 좀 더 합리적인 방향으로 아키텍쳐를 구성할 수 있다. 특히 일반 신경망과 달리, CNN의 레이어들은 가로,세로,깊이의 3개 차원을 갖게 된다 ( 여기에서 말하는 깊이란 전체 신경망의 깊이가 아니라 액티베이션 볼륨 ( activation volume ) 에서의 3번 째 차원을 이야기 함 ). 예를 들어 CIFAR-10 이미지는 32x32x3 (가로,세로,깊이) 의 차원을 갖는 입력 액티베이션 볼륨 (activation volume)이라고 볼 수 있다. 조만간 보겠지만, 하나의 레이어에 위치한 뉴런들은 일반 신경망과는 달리 앞 레이어의 전체 뉴런이 아닌 일부에만 연결이 되어 있다. CNN 아키텍쳐는 전체 이미지를 클래스 점수들로 이뤄진 하나의 벡터로 만들어주기 때문에 마지막 출력 레이어는 1x1x10(10은 CIFAR-10 데이터의 클래스 개수)의 차원을 가지게 된다. 이에 대한 그럼은 아래와 같다:

<div class="fig figcenter fighighlight">
  <img src="{{site.baseurl}}/assets/nn1/neural_net2.jpeg" width="40%">
  <img src="{{site.baseurl}}/assets/cnn/cnn.jpeg" width="48%" style="border-left: 1px solid black;">
  <div class="figcaption">좌: 일반 3-레이어 신경망. 우: 그림과 같이 CNN은 뉴런들을 3차원으로 배치한다. CNN의 모든 레이어는 3차원 입력 볼륨을 3차원 출력 볼륨으로 변환 (transform) 시킨다. 이 예제에서 붉은 색으로 나타난 입력 레이어는 이미지를 입력으로 받으므로, 이 레이어의 가로/세로/채널은 각각 이미지의 가로/세로/3(Red,Green,Blue) 이다.</div>
</div>

> CNN은 여러 레이어로 이루어져 있다. 각각의 레이어는 3차원의 볼륨을 입력으로 받고 미분 가능한 함수를 거쳐 3차원의 볼륨을 출력하는  간단한 기능을 한다.

<a name='layers'></a>

### CNN을 이루는 레이어들

위에서 다룬 것과 같이, CNN의 각 레이어는 미분 가능한 변환 함수를 통해 하나의 액티베이션 볼륨을 또다른 액티베이션 볼륨으로 변환 (transform) 시킨다. CNN 아키텍쳐에서는 크게 컨볼루셔널 레이어, 풀링 레이어, Fully-connected 레이어라는 3개 종류의 레이어가 사용된다. 전체 CNN 아키텍쳐는 이 3 종류의 레이어들을 쌓아 만들어진다.

*예제: 아래에서 더 자세하게 배우겠지만, CIFAR-10 데이터를 다루기 위한 간단한 CNN은 [INPUT-CONV-RELU-POOL-FC]로 구축할 수 있다.

- INPUT 입력 이미지가 가로32, 세로32, 그리고 RGB 채널을 가지는 경우 입력의 크기는 [32x32x3].
- CONV 레이어는 입력 이미지의 일부 영역과 연결되어 있으며, 이 연결된 영역과 자신의 가중치의 내적 연산 (dot product) 을 계산하게 된다. 결과 볼륨은 [32x32x12]와 같은 크기를 갖게 된다.
- RELU 레이어는 max(0,x)와 같이 각 요소에 적용되는 액티베이션 함수 (activation function)이다. 이 레이어는 볼륨의 크기를 변화시키지 않는다 ([32x32x12])
- POOL 레이어는 (가로,세로) 차원에 대해 다운샘플링 (downsampling)을 수행해 [16x16x12]와 같이 줄어든 볼륨을 출력한다.
- FC (fully-connected) 레이어는 클래스 점수들을 계산해 [1x1x10]의 크기를 갖는 볼륨을 출력한다. 10개 숫자들은 10개 카테고리에 대한 클래스 점수에 해당한다. 레이어의 이름에서 유추 가능하듯, 이 레이어는 이전 볼륨의 모든 요소와 연결되어 있다.

이와 같이, CNN은 픽셀 값으로 이뤄진 원본 이미지를 각 레이어를 거치며 클래스 점수로 변환 (transform) 시킨다. 한 가지 기억할 것은, 어떤 레이어는 모수 (parameter)를 갖지만 어떤 레이어는 모수를 갖지 않는다는 것이다. 특히 CONV/FC 레이어들은 단순히 입력 볼륨만이 아니라 가중치(weight)와 바이어스(bias) 또한 포함하는 액티베이션(activation) 함수이다. 반면 RELU/POOL 레이어들은 고정된 함수이다. CONV/FC 레이어의 모수 (parameter)들은 각 이미지에 대한 클래스 점수가 해당 이미지의 레이블과 같아지도록 그라디언트 디센트 (gradient descent)로 학습된다.

요약해보면:

- CNN 아키텍쳐는 여러 레이어를 통해 입력 이미지 볼륨을 출력 볼륨 ( 클래스 점수 )으로 변환시켜 준다.
- CNN은 몇 가지 종류의 레이어로 구성되어 있다. CONV/FC/RELU/POOL 레이어가 현재 가장 많이 쓰인다.
- 각 레이어는 3차원의 입력 볼륨을 미분 가능한 함수를 통해 3차원 출력 볼륨으로 변환시킨다.
- 모수(parameter)가 있는 레이어도 있고 그렇지 않은 레이어도 있다 (FC/CONV는 모수를 갖고 있고, RELU/POOL 등은 모수가 없음).
- 초모수 (hyperparameter)가 있는 레이어도 있고 그렇지 않은 레이어도 있다 (CONV/FC/POOL 레이어는 초모수를 가지며 RELU는 가지지 않음).

<div class="fig figcenter fighighlight">
  <img src="{{site.baseurl}}/assets/cnn/convnet.jpeg" width="100%">
  <div class="figcaption">
    CNN 아키텍쳐의 액티베이션 (activation) 예제. 첫 볼륨은 로우 이미지(raw image)를 다루며, 마지막 볼륨은 클래스 점수들을 출력한다. 입/출력 사이의 액티베이션들은 그림의 각 열에 나타나 있다. 3차원 볼륨을 시각적으로 나타내기가 어렵기 때문에 각 행마다 볼륨들의 일부만 나타냈다. 마지막 레이어는 모든 클래스에 대한 점수를 나타내지만 여기에서는 상위 5개 클래스에 대한 점수와 레이블만 표시했다. <a href="http://cs231n.stanford.edu/">전체 웹 데모</a>는 우리의 웹사이트 상단에 있다. 여기에서 사용된 아키텍쳐는 작은 VGG Net이다. 
  </div>
</div>

이제 각각의 레이어에 대해 초모수(hyperparameter)나 연결성 (connectivity) 등의 세부 사항들을 알아보도록 하자.

<a name='conv'></a>

#### 컨볼루셔널 레이어 (이하 CONV)

CONV 레이어는 CNN을 이루는 핵심 요소이다. CONV 레이어의 출력은 3차원으로 정렬된 뉴런들로 해석될 수 있다. 이제부터는 뉴런들의 연결성 (connectivity), 그들의 공간상의 배치, 그리고 모수 공유(parameter sharing) 에 대해 알아보자.

**개요 및 직관적인 설명.** CONV 레이어의 모수(parameter)들은 일련의 학습가능한 필터들로 이뤄져 있다. 각 필터는 가로/세로 차원으로는 작지만 깊이 (depth) 차원으로는 전체 깊이를 아우른다. 포워드 패스 (forward pass) 때에는 각 필터를 입력 볼륨의 가로/세로 차원으로 슬라이딩 시키며 (정확히는 convolve 시키며) 2차원의 액티베이션 맵 (activation map)을 생성한다. 필터를 입력 위로 슬라이딩 시킬 때, 필터와 입력의 요소들 사이의 내적 연산 (dot product)이 이뤄진다. 직관적으로 설명하면, 이 신경망은 입력의 특정 위치의 특정 패턴에 대해 반응하는 (activate) 필터를 학습한다.  이런 액티베이션 맵 (activation map)을 깊이 (depth) 차원을 따라 쌓은 것이 곧 출력 볼륨이 된다. 그러므로 출력 볼륨의 각 요소들은 입력의 작은 영역만을 취급하고, 같은 액티베이션 맵 내의 뉴런들은 같은 모수들을 공유한다 (같은 필터를 적용한 결과이므로).  이제 이 과정에 대해 좀 더 깊이 파헤쳐보자.

**로컬 연결성 (Local connectivity).** 이미지와 같은 고차원 입력을 다룰 때에는, 현재 레이어의 한 뉴런을 이전 볼륨의 모든 뉴런들과 연결하는 것이 비 실용적이다. 대신에 우리는 레이어의 각 뉴런을 입력 볼륨의 로컬한 영역(local region)에만 연결할 것이다. 이 영역은 리셉티브 필드 (receptive field)라고 불리는 초모수 (hyperparameter) 이다. 깊이 차원 측면에서는 항상 입력 볼륨의 총 깊이를 다룬다 (가로/세로는 작은 영역을 보지만 깊이는 전체를 본다는 뜻). 공간적 차원 (가로/세로)와 깊이 차원을 다루는 방식이 다르다는 걸 기억하자. 

*예제 1*. 예를 들어 입력 볼륨의 크기가 (CIFAR-10의 RGB 이미지와 같이) [32x32x3]이라고 하자. 만약 리셉티브 필드의 크기가 5x5라면, CONV 레이어의 각 뉴런은 입력 볼륨의 [5x5x3] 크기의 영역에 가중치 (weight)를 가하게 된다 (총 5x5x3=75 개 가중치). 입력 볼륨 (RGB 이미지)의 깊이가 3이므로 마지막 숫자가 3이 된다는 것을 기억하자. 

*예제 2*. 입력 볼륨의 크기가 [16x16x20]이라고 하자. 3x3 크기의 리셉티브 필드를 사용하면 CONV 레이어의 각 뉴런은 입력 볼륨과 3x3x20=180 개의 연결을 갖게 된다. 이번에도 입력 볼륨의 깊이가 20이므로 마지막 숫자가 20이 된다는 것을 기억하자.

<div class="fig figcenter fighighlight">
  <img src="{{site.baseurl}}/assets/cnn/depthcol.jpeg" width="40%">
  <img src="{{site.baseurl}}/assets/nn1/neuron_model.jpeg" width="40%" style="border-left: 1px solid black;">
  <div class="figcaption">
    <b>좌:</b> 입력 볼륨(붉은색, 32x32x3 크기의 CIFAR-10 이미지)과 첫번째 컨볼루션 레이어 볼륨. 컨볼루션 레이어의 각 뉴런은 입력 볼륨의 일부 영역에만 연결된다 (가로/세로 공간 차원으로는 일부 연결, 깊이(컬러 채널) 차원은 모두 연결). 컨볼루션 레이어의 깊이 차원의 여러 뉴런 (그림에서 5개)들이 모두 입력의 같은 영역을 처리한다는 것을 기억하자 (깊이 차원과 관련해서는 아래에서 더 자세히 알아볼 것임). 우: 입력의 일부 영역에만 연결된다는 점을 제외하고는, 이전 신경망 챕터에서 다뤄지던 뉴런들과 똑같이 내적 연산과 비선형 함수로 이뤄진다.
  </div>
</div>

**공간적 배치**. 지금까지는 컨볼루션 레이어의 한 뉴런과 입력 볼륨의 연결에 대해 알아보았다. 그러나 아직 출력 볼륨에 얼마나 많은 뉴런들이 있는지, 그리고 그 뉴런들이 어떤식으로 배치되는지는 다루지 않았다. 3개의 hyperparameter들이 출력 볼륨의 크기를 결정하게 된다. 그 3개 요소는 바로 **깊이, stride, 그리고 제로 패딩 (zero-padding)** 이다. 이들에 대해 알아보자:

1. 먼저, 출력 볼륨의 **깊이** 는 우리가 결정할 수 있는 요소이다. 컨볼루션 레이어의 뉴런들 중 입력 볼륨 내 동일한 영역과 연결된 뉴런의 개수를 의미한다. 마치 일반 신경망에서 히든 레이어 내의 모든 뉴런들이 같은 입력값과 연결된 것과 비슷하다. 앞으로 살펴보겠지만, 이 뉴런들은 입력에 대해 서로 다른 특징 (feature)에 활성화된다 (activate). 예를 들어, 이미지를 입력으로 받는 첫 번째 컨볼루션 레이어의 경우, 깊이 축에 따른 각 뉴런들은 이미지의 서로 다른 엣지, 색깔, 블롭(blob) 등에 활성화된다. 앞으로는 인풋의 서로 같은 영역을 바라보는 뉴런들을 **깊이 컬럼 (depth column)**이라고 부르겠다.
2. 두 번째로 어떤 간격 (가로/세로의 공간적 간격) 으로 깊이 컬럼을 할당할 지를 의미하는 **stride**를 결정해야 한다. 만약 stride가 1이라면, 깊이 컬럼을 1칸마다 할당하게 된다 (한 칸 간격으로 깊이 컬럼 할당). 이럴 경우 각 깊이 컬럼들은 receptive field 상 넓은 영역이 겹치게 되고, 출력 볼륨의 크기도 매우 커지게 된다. 반대로, 큰 stride를 사용한다면 receptive field끼리 좁은 영역만 겹치게 되고 출력 볼륨도 작아지게 된다 (깊이는 작아지지 않고 가로/세로만 작아지게 됨).
3. 조만간 살펴보겠지만, 입력 볼륨의 가장자리를 0으로 패딩하는 것이 좋을 때가 있다. 이 **zero-padding**은 hyperparamter이다. zero-padding을 사용할 때의 장점은, 출력 볼륨의 공간적 크기(가로/세로)를 조절할 수 있다는 것이다. 특히 입력 볼륨의 공간적 크기를 유지하고 싶은 경우 (입력의 가로/세로 = 출력의 가로/세로) 사용하게 된다.

출력 볼륨의 공간적 크기 (가로/세로)는 입력 볼륨 크기 ($$W$$), CONV 레이어의 리셉티브 필드 크기($$F$$)와 stride ($$S$$), 그리고 제로 패딩 (zero-padding) 사이즈 ($$P$$) 의 함수로 계산할 수 있다. $$(W - F + 2P)/S + 1$$. I을 통해 알맞은 크기를 계산하면 된다. 만약 이 값이 정수가 아니라면 stride가 잘못 정해진 것이다. 이 경우 뉴런들이 대칭을 이루며 깔끔하게 배치되는 것이 불가능하다. 다음 예제를 보면 이 수식을 좀 더 직관적으로 이해할 수 있을 것이다:

<div class="fig figcenter fighighlight">
  <img src="{{site.baseurl}}/assets/cnn/stride.jpeg">
  <div class="figcaption">
  공간적 배치에 관한 그림. 이 예제에서는 가로/세로 공간적 차원 중 하나만 고려한다 (x축). 리셉티브 필드 F=3, 입력 사이즈 W=5, 제로 패딩 P=1. <b>좌</b>: 뉴런들이 stride S=1을 갖고 배치된 경우,  출력 사이즈는 (5-3+2)/1 +1 = 5이다. <b>우</b>: stride S=2인 경우 (5-3+2)/2 + 1 = 3의 출력 사이즈를 가진다. Stride S=3은 사용할 수 없다. (5-3+2) = 4가 3으로 나눠지지 않기 때문에 출력 볼륨의 뉴런들이 깔끔히 배치되지 않는다. 
  이 예에서 뉴런들의 가중치는 [1,0,-1] (가장 오른쪽) 이며 bias는 0이다. 이 가중치는 노란 뉴런들 모두에게 공유된다 (아래에서 parameter sharing에 대해 살펴보라).
  </div>
</div>

*제로 패딩 사용*. 위 예제의 왼쪽 그림에서, 입력과 출력의 차원이 모두 5라는 것을 기억하자. 리셉티브 필드가 3이고 제로 패딩이 1이기 때문에 이런 결과가 나오는 것이다. 만약 제로 패딩이 사용되지 않았다면 출력 볼륨의 크기는 3이 될 것이다. 일반적으로, 제로 패딩을 $$P = (F - 1)/2$$ , stride $$S = 1$$로 세팅하면 입/출력의 크기가 같아지게 된다. 이런 방식으로 사용하는 것이 일반적이며, 앞으로 컨볼루션 신경망에 대해 다루면서 그 이유에 대해 더 알아볼 것이다.

*Stride에 대한 constraints*. 공간적 배치와 관련된 hyperparameter들은 상호 constraint들이 존재한다는 것을 기억하자. 예를 들어, 입력 사이즈 $$W=10$$이고 제로 패딩이 사용되지 않았고 $$P=0$$, 필터 사이즈가 $$F=3$$이라면, stride $$S=2$$를 사용하는 것이 불가능하다. $$(W - F + 2P)/S + 1 = (10 - 3 + 0) / 2 + 1 = 4.5$$이 정수가 아니기 때문이다. 그러므로 hyperparameter를 이런 식으로 설정하면 컨볼루션 신경망 관련 라이브러리들은 exception을 낸다. 컨볼루션 신경망의 구조 관련 섹션에서 확인하겠지만, 전체 신경망이 잘 돌아가도록 이런 숫자들을 설정하는 과정은 매우 골치 아프다. 제로 패딩이나 다른 신경망 디자인 비법들을 사용하면 훨씬 수월하게 진행할 수 있다.

*실제 예제*. 이미지넷 대회에서 우승한 [Krizhevsky et al.](http://papers.nips.cc/paper/4824-imagenet-classification-with-deep-convolutional-neural-networks) 의 모델의 경우 [227x227x3] 크기의 이미지를 입력으로 받는다. 첫 번째 컨볼루션 레이어에서는 리셉티브 필드 $$F=11$$, stride $$S=4$$를 사용했고 제로 패딩은 사용하지 않았다 $$P=0$$. (227 - 11)/4 +1=55 이고 컨볼루션 레이어의 깊이는 $$K=96$$이므로 이 컨볼루션 레이어의 크기는 [55x55x96]이 된다. 각각의 55\*55\*96개 뉴런들은 입력 볼륨의 [11x11x3]개 뉴런들과 연결되어 있다. 그리고 각 깊이의 모든 96개 뉴런들은 입력 볼륨의 같은 [11x11x3] 영역에 서로 다른 가중치를 가지고 연결된다.

**파라미터 공유**. 파라미터 공유 기법은 컨볼루션 레이어의 파라미터 개수를 조절하기 위해 사용된다. 위의 실제 예제에서 보았듯, 첫 번째 컨볼루션 레이어에는 55\*55\*96 = 290,400 개의 뉴런이 있고 각각의 뉴런은 11\*11\*3 = 363개의 가중치와 1개의 바이어스를 가진다. 첫 번째 컨볼루션 레이어만 따져도 총 파라미터 개수는  290400*364=105,705,600개가 된다. 분명히 이 숫자는 너무 크다.

사실 적절한 가정을 통해 파라미터 개수를 크게 줄이는 것이 가능하다: (x,y)에서 어떤 patch feature가 유용하게 사용되었다면, 이 feature는 다른 위치 (x2,y2)에서도 유용하게 사용될 수 있다. 3차원 볼륨의 한 슬라이스 (깊이 차원으로 자른 2차원 슬라이스) 를 **depth slice**라고 하자 ([55x55x96] 사이즈의 볼륨은 각각 [55x55]의 크기를 가진 96개의 depth slice임). 앞으로는 각 depth slice 내의 뉴런들이 같은 가중치와 바이어스를 가지도록 제한할 것이다. 이런 파라미터 공유 기법을 사용하면, 예제의 첫 번째 컨볼루션 레이어는 (depth slice 당) 96개의 고유한 가중치를 가져서 총 96\*11\*11\*3 = 34,848개의 고유한 가중치, 또는 바이어스를 합쳐서 34,944개의 파라미터를 갖게 된다. 또는 각 depth slice에 존재하는 55*55개의 뉴런들은 모두 같은 파라미터를 사용하게 된다. 실제로는 backpropagation 과정에서 각 depth slice 내의 모든 뉴런들이 가중치에 대한 gradient를 계산하겠지만, 가중치 업데이트 할 때에는 이 gradient들을 합해 사용한다. 

Notice that if all neurons in a single depth slice are using the same weight vector, then the forward pass of the CONV layer can in each depth slice be computed as a **convolution** of the neuron's weights with the input volume (Hence the name: Convolutional Layer). Therefore, it is common to refer to the sets of weights as a **filter** (or a **kernel**), which is convolved with the input. The result of this convolution is an *activation map* (e.g. of size [55x55]), and the set of activation maps for each different filter are stacked together along the depth dimension to produce the output volume (e.g. [55x55x96]).

<div class="fig figcenter fighighlight">
  <img src="{{site.baseurl}}/assets/cnn/weights.jpeg">
  <div class="figcaption">
    Example filters learned by Krizhevsky et al. Each of the 96 filters shown here is of size [11x11x3], and each one is shared by the 55*55 neurons in one depth slice. Notice that the parameter sharing assumption is relatively reasonable: If detecting a horizontal edge is important at some location in the image, it should intuitively be useful at some other location as well due to the translationally-invariant structure of images. There is therefore no need to relearn to detect a horizontal edge at every one of the 55*55 distinct locations in the Conv layer output volume.
  </div>
</div>

Note that sometimes the parameter sharing assumption may not make sense. This is especially the case when the input images to a ConvNet have some specific centered structure, where we should expect, for example, that completely different features should be learned on one side of the image than another. One practical example is when the input are faces that have been centered in the image. You might expect that different eye-specific or hair-specific features could (and should) be learned in different spatial locations. In that case it is common to relax the parameter sharing scheme, and instead simply call the layer a **Locally-Connected Layer**.

**Numpy examples.** To make the discussion above more concrete, lets express the same ideas but in code and with a specific example. Suppose that the input volume is a numpy array `X`. Then:

- A *depth column* at position `(x,y)` would be the activations `X[x,y,:]`.
- A *depth slice*, or equivalently an *activation map* at depth `d` would be the activations `X[:,:,d]`.

*Conv Layer Example*. Suppose that the input volume `X` has shape `X.shape: (11,11,4)`. Suppose further that we use no zero padding ($$P = 0$$), that the filter size is $$F = 5$$, and that the stride is $$S = 2$$. The output volume would therefore have spatial size (11-5)/2+1 = 4, giving a volume with width and height of 4. The activation map in the output volume (call it `V`), would then look as follows (only some of the elements are computed in this example):

- `V[0,0,0] = np.sum(X[:5,:5,:] * W0) + b0`
- `V[1,0,0] = np.sum(X[2:7,:5,:] * W0) + b0`
- `V[2,0,0] = np.sum(X[4:9,:5,:] * W0) + b0`
- `V[3,0,0] = np.sum(X[6:11,:5,:] * W0) + b0`

Remember that in numpy, the operation `*` above denotes elementwise multiplication between the arrays. Notice also that the weight vector `W0` is the weight vector of that neuron and `b0` is the bias. Here, `W0` is assumed to be of shape `W0.shape: (5,5,4)`, since the filter size is 5 and the depth of the input volume is 4. Notice that at each point, we are computing the dot product as seen before in ordinary neural networks. Also, we see that we are using the same weight and bias (due to parameter sharing), and where the dimensions along the width are increasing in steps of 2 (i.e. the stride). To construct a second activation map in the output volume, we would have:

- `V[0,0,1] = np.sum(X[:5,:5,:] * W1) + b1`
- `V[1,0,1] = np.sum(X[2:7,:5,:] * W1) + b1`
- `V[2,0,1] = np.sum(X[4:9,:5,:] * W1) + b1`
- `V[3,0,1] = np.sum(X[6:11,:5,:] * W1) + b1`
- `V[0,1,1] = np.sum(X[:5,2:7,:] * W1) + b1` (example of going along y)
- `V[2,3,1] = np.sum(X[4:9,6:11,:] * W1) + b1` (or along both)

where we see that we are indexing into the second depth dimension in `V` (at index 1) because we are computing the second activation map, and that a different set of parameters (`W1`) is now used. In the example above, we are for brevity leaving out some of the other operatations the Conv Layer would perform to fill the other parts of the output array `V`. Additionally, recall that these activation maps are often followed elementwise through an activation function such as ReLU, but this is not shown here.

**Summary**. To summarize, the Conv Layer:

- Accepts a volume of size $$W_1 \times H_1 \times D_1$$
- Requires four hyperparameters:
  - Number of filters $$K$$,
  - their spatial extent $$F$$,
  - the stride $$S$$,
  - the amount of zero padding $$P$$.
- Produces a volume of size $$W_2 \times H_2 \times D_2$$ where:
  - $$W_2 = (W_1 - F + 2P)/S + 1$$
  - $$H_2 = (H_1 - F + 2P)/S + 1$$ (i.e. width and height are computed equally by symmetry)
  - $$D_2 = K$$
- With parameter sharing, it introduces $$F \cdot F \cdot D_1$$ weights per filter, for a total of $$(F \cdot F \cdot D_1) \cdot K$$ weights and $$K$$ biases.
- In the output volume, the $$d$$-th depth slice (of size $$W_2 \times H_2$$) is the result of performing a valid convolution of the $$d$$-th filter over the input volume with a stride of $$S$$, and then offset by $$d$$-th bias.

A common setting of the hyperparameters is $$F = 3, S = 1, P = 1$$. However, there are common conventions and rules of thumb that motivate these hyperparameters. See the [ConvNet architectures](#architectures) section below.

**Convolution Demo**. Below is a running demo of a CONV layer. Since 3D volumes are hard to visualize, all the volumes (the input volume (in blue), the weight volumes (in red), the output volume (in green)) are visualized with each depth slice stacked in rows. The input volume is of size $$W_1 = 5, H_1 = 5, D_1 = 3$$, and the CONV layer parameters are $$K = 2, F = 3, S = 2, P = 1$$. That is, we have two filters of size $$3 \times 3$$, and they are applied with a stride of 2. Therefore, the output volume size has spatial size (5 - 3 + 2)/2 + 1 = 3. Moreover, notice that a padding of $$P = 1$$ is applied to the input volume, making the outer border of the input volume zero. The visualization below iterates over the output activations (green), and shows that each element is computed by elementwise multiplying the highlighted input (blue) with the filter (red), summing it up, and then offsetting the result by the bias.

<div class="fig figcenter fighighlight">
  <iframe src="{{site.baseurl}}/assets/conv-demo/index.html" width="100%" height="700px;" style="border:none;"></iframe>
  <div class="figcaption"></div>
</div>

**Implementation as Matrix Multiplication**. Note that the convolution operation essentially performs dot products between the filters and local regions of the input. A common implementation pattern of the CONV layer is to take advantage of this fact and formulate the forward pass of a convolutional layer as one big matrix multiply as follows:

1. The local regions in the input image are stretched out into columns in an operation commonly called **im2col**. For example, if the input is [227x227x3] and it is to be convolved with 11x11x3 filters at stride 4, then we would take [11x11x3] blocks of pixels in the input and stretch each block into a column vector of size 11\*11\*3 = 363. Iterating this process in the input at stride of 4 gives (227-11)/4+1 = 55 locations along both width and height, leading to an output matrix `X_col` of *im2col* of size [363 x 3025], where every column is a stretched out receptive field and there are 55*55 = 3025 of them in total. Note that since the receptive fields overlap, every number in the input volume may be duplicated in multiple distinct columns.
2. The weights of the CONV layer are similarly stretched out into rows. For example, if there are 96 filters of size [11x11x3] this would give a matrix `W_row` of size [96 x 363].
3. The result of a convolution is now equivalent to performing one large matrix multiply `np.dot(W_row, X_col)`, which evaluates the dot product between every filter and every receptive field location. In our example, the output of this operation would be [96 x 3025], giving the output of the dot product of each filter at each location.
4. The result must finally be reshaped back to its proper output dimension [55x55x96].

This approach has the downside that it can use a lot of memory, since some values in the input volume are replicated multiple times in `X_col`. However, the benefit is that there are many very efficient implementations of Matrix Multiplication that we can take advantage of (for example, in the commonly used [BLAS](http://www.netlib.org/blas/) API). Morever, the same *im2col* idea can be reused to perform the pooling operation, which we discuss next.

**Backpropagation.** The backward pass for a convolution operation (for both the data and the weights) is also a convolution (but with spatially-flipped filters). This is easy to derive in the 1-dimensional case with a toy example (not expanded on for now).

<a name='pool'></a>
#### Pooling Layer

It is common to periodically insert a Pooling layer in-between successive Conv layers in a ConvNet architecture. Its function is to progressively reduce the spatial size of the representation to reduce the amount of parameters and computation in the network, and hence to also control overfitting. The Pooling Layer operates independently on every depth slice of the input and resizes it spatially, using the MAX operation. The most common form is a pooling layer with filters of size 2x2 applied with a stride of 2 downsamples every depth slice in the input by 2 along both width and height, discarding 75% of the activations. Every MAX operation would in this case be taking a max over 4 numbers (little 2x2 region in some depth slice). The depth dimension remains unchanged. More generally, the pooling layer:

- Accepts a volume of size $$W_1 \times H_1 \times D_1$$
- Requires three hyperparameters:
  - their spatial extent $$F$$,
  - the stride $$S$$,
- Produces a volume of size $$W_2 \times H_2 \times D_2$$ where:
  - $$W_2 = (W_1 - F)/S + 1$$
  - $$H_2 = (H_1 - F)/S + 1$$
  - $$D_2 = D_1$$
- Introduces zero parameters since it computes a fixed function of the input
- Note that it is not common to use zero-padding for Pooling layers

It is worth noting that there are only two commonly seen variations of the max pooling layer found in practice: A pooling layer with $$F = 3, S = 2$$ (also called overlapping pooling), and more commonly $$F = 2, S = 2$$. Pooling sizes with larger receptive fields are too destructive.

**General pooling**. In addition to max pooling, the pooling units can also perform other functions, such as *average pooling* or even *L2-norm pooling*. Average pooling was often used historically but has recently fallen out of favor compared to the max pooling operation, which has been shown to work better in practice.

<div class="fig figcenter fighighlight">
  <img src="{{site.baseurl}}/assets/cnn/pool.jpeg" width="36%">
  <img src="{{site.baseurl}}/assets/cnn/maxpool.jpeg" width="59%" style="border-left: 1px solid black;">
  <div class="figcaption">
    Pooling layer downsamples the volume spatially, independently in each depth slice of the input volume. <b>Left:</b> In this example, the input volume of size [224x224x64] is pooled with filter size 2, stride 2 into output volume of size [112x112x64]. Notice that the volume depth is preserved. <b>Right:</b> The most common downsampling operation is max, giving rise to <b>max pooling</b>, here shown with a stride of 2. That is, each max is taken over 4 numbers (little 2x2 square).
  </div>
</div>

**Backpropagation**. Recall from the backpropagation chapter that the backward pass for a max(x, y) operation has a simple interpretation as only routing the gradient to the input that had the highest value in the forward pass. Hence, during the forward pass of a pooling layer it is common to keep track of the index of the max activation (sometimes also called *the switches*) so that gradient routing is efficient during backpropagation.

**Recent developments**.

- [Fractional Max-Pooling](http://arxiv.org/abs/1412.6071) suggests a method for performing the pooling operation with filters smaller than 2x2. This is done by randomly generating pooling regions with a combination of 1x1, 1x2, 2x1 or 2x2 filters to tile the input activation map. The grids are generated randomly on each forward pass, and at test time the predictions can be averaged across several grids.
- [Striving for Simplicity: The All Convolutional Net](http://arxiv.org/abs/1412.6806) proposes to discard the pooling layer in favor of architecture that only consists of repeated CONV layers. To reduce the size of the representation they suggest using larger stride in CONV layer once in a while.

Due to the aggressive reduction in the size of the representation (which is helpful only for smaller datasets to control overfitting), the trend in the literature is towards discarding the pooling layer in modern ConvNets.

<a name='norm'></a>
#### Normalization Layer

Many types of normalization layers have been proposed for use in ConvNet architectures, sometimes with the intentions of implementing inhibition schemes observed in the biological brain. However, these layers have recently fallen out of favor because in practice their contribution has been shown to be minimal, if any. For various types of normalizations, see the discussion in Alex Krizhevsky's [cuda-convnet library API](http://code.google.com/p/cuda-convnet/wiki/LayerParams#Local_response_normalization_layer_(same_map)).

<a name='fc'></a>
#### Fully-connected layer

Neurons in a fully connected layer have full connections to all activations in the previous layer, as seen in regular Neural Networks. Their activations can hence be computed with a matrix multiplication followed by a bias offset. See the *Neural Network* section of the notes for more information.

<a name='convert'></a>
#### Converting FC layers to CONV layers

It is worth noting that the only difference between FC and CONV layers is that the neurons in the CONV layer are connected only to a local region in the input, and that many of the neurons in a CONV volume share parameters. However, the neurons in both layers still compute dot products, so their functional form is identical. Therefore, it turns out that it's possible to convert between FC and CONV layers:

- For any CONV layer there is an FC layer that implements the same forward function. The weight matrix would be a large matrix that is mostly zero except for at certian blocks (due to local connectivity) where the weights in many of the blocks are equal (due to parameter sharing).
- Conversely, any FC layer can be converted to a CONV layer. For example, an FC layer with $$K = 4096$$ that is looking at some input volume of size $$7 \times 7 \times 512$$ can be equivalently expressed as a CONV layer with $$F = 7, P = 0, S = 1, K = 4096$$. In other words, we are setting the filter size to be exactly the size of the input volume, and hence the output will simply be $$1 \times 1 \times 4096$$ since only a single depth column "fits" across the input volume, giving identical result as the initial FC layer.

**FC->CONV conversion**. Of these two conversions, the ability to convert an FC layer to a CONV layer is particularly useful in practice. Consider a ConvNet architecture that takes a 224x224x3 image, and then uses a series of CONV layers and POOL layers to reduce the image to an activations volume of size 7x7x512 (in an *AlexNet* architecture that we'll see later, this is done by use of 5 pooling layers that downsample the input spatially by a factor of two each time, making the final spatial size 224/2/2/2/2/2 = 7). From there, an AlexNet uses two FC layers of size 4096 and finally the last FC layers with 1000 neurons that compute the class scores. We can convert each of these three FC layers to CONV layers as described above:

- Replace the first FC layer that looks at [7x7x512] volume with a CONV layer that uses filter size $$F = 7$$, giving output volume [1x1x4096].
- Replace the second FC layer with a CONV layer that uses filter size $$F = 1$$, giving output volume [1x1x4096]
- Replace the last FC layer similarly, with $$F=1$$, giving final output [1x1x1000]

Each of these conversions could in practice involve manipulating (e.g. reshaping) the weight matrix $$W$$ in each FC layer into CONV layer filters. It turns out that this conversion allows us to "slide" the original ConvNet very efficiently across many spatial positions in a larger image, in a single forward pass.

For example, if 224x224 image gives a volume of size [7x7x512] - i.e. a reduction by 32, then forwarding an image of size 384x384 through the converted architecture would give the equivalent volume in size [12x12x512], since 384/32 = 12. Following through with the next 3 CONV layers that we just converted from FC layers would now give the final volume of size [6x6x1000], since (12 - 7)/1 + 1 = 6. Note that instead of a single vector of class scores of size [1x1x1000], we're now getting and entire 6x6 array of class scores across the 384x384 image.

> Evaluating the original ConvNet (with FC layers) independently across 224x224 crops of the 384x384 image in strides of 32 pixels gives an identical result to forwarding the converted ConvNet one time.

Naturally, forwarding the converted ConvNet a single time is much more efficient than iterating the original ConvNet over all those 36 locations, since the 36 evaluations share computation. This trick is often used in practice to get better performance, where for example, it is common to resize an image to make it bigger, use a converted ConvNet to evaluate the class scores at many spatial positions and then average the class scores.

Lastly, what if we wanted to efficiently apply the original ConvNet over the image but at a stride smaller than 32 pixels? We could achieve this with multiple forward passes. For example, note that if we wanted to use a stride of 16 pixels we could do so by combining the volumes received by forwarding the converted ConvNet twice: First over the original image and second over the image but with the image shifted spatially by 16 pixels along both width and height.

- An IPython Notebook on [Net Surgery](https://github.com/BVLC/caffe/blob/master/examples/net_surgery.ipynb) shows how to perform the conversion in practice, in code (using Caffe)

<a name='architectures'></a>
### ConvNet Architectures

We have seen that Convolutional Networks are commonly made up of only three layer types: CONV, POOL (we assume Max pool unless stated otherwise) and FC (short for fully-connected). We will also explicitly write the RELU activation function as a layer, which applies elementwise non-linearity. In this section we discuss how these are commonly stacked together to form entire ConvNets.

<a name='layerpat'></a>
#### Layer Patterns
The most common form of a ConvNet architecture stacks a few CONV-RELU layers, follows them with POOL layers, and repeats this pattern until the image has been merged spatially to a small size. At some point, it is common to transition to fully-connected layers. The last fully-connected layer holds the output, such as the class scores. In other words, the most common ConvNet architecture follows the pattern:

`INPUT -> [[CONV -> RELU]*N -> POOL?]*M -> [FC -> RELU]*K -> FC`

where the `*` indicates repetition, and the `POOL?` indicates an optional pooling layer. Moreover, `N >= 0` (and usually `N <= 3`), `M >= 0`, `K >= 0` (and usually `K < 3`). For example, here are some common ConvNet architectures you may see that follow this pattern:

- `INPUT -> FC`, implements a linear classifier. Here `N = M = K = 0`.
- `INPUT -> CONV -> RELU -> FC`
- `INPUT -> [CONV -> RELU -> POOL]*2 -> FC -> RELU -> FC`. Here we see that there is a single CONV layer between every POOL layer.
- `INPUT -> [CONV -> RELU -> CONV -> RELU -> POOL]*3 -> [FC -> RELU]*2 -> FC` Here we see two CONV layers stacked before every POOL layer. This is generally a good idea for larger and deeper networks, because multiple stacked CONV layers can develop more complex features of the input volume before the destructive pooling operation.

*Prefer a stack of small filter CONV to one large receptive field CONV layer*. Suppose that you stack three 3x3 CONV layers on top of each other (with non-linearities in between, of course). In this arrangement, each neuron on the first CONV layer has a 3x3 view of the input volume. A neuron on the second CONV layer has a 3x3 view of the first CONV layer, and hence by extension a 5x5 view of the input volume. Similarly, a neuron on the third CONV layer has a 3x3 view of the 2nd CONV layer, and hence a 7x7 view of the input volume. Suppose that instead of these three layers of 3x3 CONV, we only wanted to use a single CONV layer with 7x7 receptive fields. These neurons would have a receptive field size of the input volume that is identical in spatial extent (7x7), but with several disadvantages. First, the neurons would be computing a linear function over the input, while the three stacks of CONV layers contain non-linearities that make their features more expressive. Second, if we suppose that all the volumes have $$C$$ channels, then it can be seen that the single 7x7 CONV layer would contain $$C \times (7 \times 7 \times C) = 49 C^2$$ parameters, while the three 3x3 CONV layers would only contain $$3 \times (C \times (3 \times 3 \times C)) = 27 C^2$$ parameters. Intuitively, stacking CONV layers with tiny filters as opposed to having one CONV layer with big filters allows us to express more powerful features of the input, and with fewer parameters. As a practical disadvantage, we might need more memory to hold all the intermediate CONV layer results if we plan to do backpropagation.

<a name='layersizepat'></a>
#### Layer Sizing Patterns

Until now we've omitted mentions of common hyperparameters used in each of the layers in a ConvNet. We will first state the common rules of thumb for sizing the architectures and then follow the rules with a discussion of the notation:

The **input layer** (that contains the image) should be divisible by 2 many times. Common numbers include 32 (e.g. CIFAR-10), 64, 96 (e.g. STL-10), or 224 (e.g. common ImageNet ConvNets), 384, and 512.

The **conv layers** should be using small filters (e.g. 3x3 or at most 5x5), using a stride of $$S = 1$$, and crucially, padding the input volume with zeros in such way that the conv layer does not alter the spatial dimensions of the input. That is, when $$F = 3$$, then using $$P = 1$$ will retain the original size of the input. When $$F = 5$$, $$P = 2$$. For a general $$F$$, it can be seen that $$P = (F - 1) / 2$$ preserves the input size. If you must use bigger filter sizes (such as 7x7 or so), it is only common to see this on the very first conv layer that is looking at the input image.

The **pool layers** are in charge of downsampling the spatial dimensions of the input. The most common setting is to use max-pooling with 2x2 receptive fields (i.e. $$F = 2$$), and with a stride of 2 (i.e. $$S = 2$$). Note that this discards exactly 75% of the activations in an input volume (due to downsampling by 2 in both width and height). Another sligthly less common setting is to use 3x3 receptive fields with a stride of 2, but this makes. It is very uncommon to see receptive field sizes for max pooling that are larger than 3 because the pooling is then too lossy and agressive. This usually leads to worse performance.

*Reducing sizing headaches.* The scheme presented above is pleasing because all the CONV layers preserve the spatial size of their input, while the POOL layers alone are in charge of down-sampling the volumes spatially. In an alternative scheme where we use strides greater than 1 or don't zero-pad the input in CONV layers, we would have to very carefully keep track of the input volumes throughout the CNN architecture and make sure that all strides and filters "work out", and that the ConvNet architecture is nicely and symmetrically wired.

*Why use stride of 1 in CONV?* Smaller strides work better in practice. Additionally, as already mentioned stride 1 allows us to leave all spatial down-sampling to the POOL layers, with the CONV layers only transforming the input volume depth-wise.

*Why use padding?* In addition to the aforementioned benefit of keeping the spatial sizes constant after CONV, doing this actually improves performance. If the CONV layers were to not zero-pad the inputs and only perform valid convolutions, then the size of the volumes would reduce by a small amount after each CONV, and the information at the borders would be "washed away" too quickly.

*Compromising based on memory constraints.* In some cases (especially early in the ConvNet architectures), the amount of memory can build up very quickly with the rules of thumb presented above. For example, filtering a 224x224x3 image with three 3x3 CONV layers with 64 filters each and padding 1 would create three activation volumes of size [224x224x64]. This amounts to a total of about 10 million activations, or 72MB of memory (per image, for both activations and gradients). Since GPUs are often bottlenecked by memory, it may be necessary to compromise. In practice, people prefer to make the compromise at only the first CONV layer of the network. For example, one compromise might be to use a first CONV layer with filter sizes of 7x7 and stride of 2 (as seen in a ZF net). As another example, an AlexNet uses filer sizes of 11x11 and stride of 4.

<a name='case'></a>
#### Case studies

There are several architectures in the field of Convolutional Networks that have a name. The most common are:

- **LeNet**. The first successful applications of Convolutional Networks were developed by Yann LeCun in 1990's. Of these, the best known is the [LeNet](http://yann.lecun.com/exdb/publis/pdf/lecun-98.pdf) architecture that was used to read zip codes, digits, etc.
- **AlexNet**. The first work that popularized Convolutional Networks in Computer Vision was the [AlexNet](http://papers.nips.cc/paper/4824-imagenet-classification-with-deep-convolutional-neural-networks), developed by Alex Krizhevsky, Ilya Sutskever and Geoff Hinton. The AlexNet was submitted to the [ImageNet ILSVRC challenge](http://www.image-net.org/challenges/LSVRC/2014/) in 2012 and significantly outperformed the second runner-up (top 5 error of 16% compared to runner-up with 26% error). The Network had a similar architecture basic as LeNet, but was deeper, bigger, and featured Convolutional Layers stacked on top of each other (previously it was common to only have a single CONV layer immediately followed by a POOL layer).
- **ZF Net**. The ILSVRC 2013 winner was a Convolutional Network from Matthew Zeiler and Rob Fergus. It became known as the [ZFNet](http://arxiv.org/abs/1311.2901) (short for Zeiler & Fergus Net). It was an improvement on AlexNet by tweaking the architecture hyperparameters, in particular by expanding the size of the middle convolutional layers.
- **GoogLeNet**. The ILSVRC 2014 winner was a Convolutional Network from [Szegedy et al.](http://arxiv.org/abs/1409.4842) from Google. Its main contribution was the development of an *Inception Module* that dramatically reduced the number of parameters in the network (4M, compared to AlexNet with 60M). Additionally, this paper uses Average Pooling instead of Fully Connected layers at the top of the ConvNet, eliminating a large amount of parameters that do not seem to matter much.
- **VGGNet**. The runner-up in ILSVRC 2014 was the network from Karen Simonyan and Andrew Zisserman that became known as the [VGGNet](http://www.robots.ox.ac.uk/~vgg/research/very_deep/). Its main contribution was in showing that the depth of the network is a critical component for good performance. Their final best network contains 16 CONV/FC layers and, appealingly, features an extremely homogeneous architecture that only performs 3x3 convolutions and 2x2 pooling from the beginning to the end. It was later found that despite its slightly weaker classification performance, the VGG ConvNet features outperform those of GoogLeNet in multiple transfer learning tasks. Hence, the VGG network is currently the most preferred choice in the community when extracting CNN features from images. In particular, their [pretrained model](http://www.robots.ox.ac.uk/~vgg/research/very_deep/) is available for plug and play use in Caffe. A downside of the VGGNet is that it is more expensive to evaluate and uses a lot more memory and parameters (140M).
- **ResNet**. [Residual Network](http://arxiv.org/abs/1512.03385) developed by Kaiming He et al. was the winner of ILSVRC 2015. It features an interesting architecture with special *skip connections* and features heavy use of batch normalization. The architecture is also missing fully connected layers at the end of the network. The reader is also referred to Kaiming's presentation ([video](https://www.youtube.com/watch?v=1PGLj-uKT1w), [slides](http://research.microsoft.com/en-us/um/people/kahe/ilsvrc15/ilsvrc2015_deep_residual_learning_kaiminghe.pdf)), and some [recent experiments](https://github.com/gcr/torch-residual-networks) that reproduce these networks in Torch.

**VGGNet in detail**.
Lets break down the [VGGNet](http://www.robots.ox.ac.uk/~vgg/research/very_deep/) in more detail. The whole VGGNet is composed of CONV layers that perform 3x3 convolutions with stride 1 and pad 1, and of POOL layers that perform 2x2 max pooling with stride 2 (and no padding). We can write out the size of the representation at each step of the processing and keep track of both the representation size and the total number of weights:

~~~
INPUT: [224x224x3]        memory:  224*224*3=150K   weights: 0
CONV3-64: [224x224x64]  memory:  224*224*64=3.2M   weights: (3*3*3)*64 = 1,728
CONV3-64: [224x224x64]  memory:  224*224*64=3.2M   weights: (3*3*64)*64 = 36,864
POOL2: [112x112x64]  memory:  112*112*64=800K   weights: 0
CONV3-128: [112x112x128]  memory:  112*112*128=1.6M   weights: (3*3*64)*128 = 73,728
CONV3-128: [112x112x128]  memory:  112*112*128=1.6M   weights: (3*3*128)*128 = 147,456
POOL2: [56x56x128]  memory:  56*56*128=400K   weights: 0
CONV3-256: [56x56x256]  memory:  56*56*256=800K   weights: (3*3*128)*256 = 294,912
CONV3-256: [56x56x256]  memory:  56*56*256=800K   weights: (3*3*256)*256 = 589,824
CONV3-256: [56x56x256]  memory:  56*56*256=800K   weights: (3*3*256)*256 = 589,824
POOL2: [28x28x256]  memory:  28*28*256=200K   weights: 0
CONV3-512: [28x28x512]  memory:  28*28*512=400K   weights: (3*3*256)*512 = 1,179,648
CONV3-512: [28x28x512]  memory:  28*28*512=400K   weights: (3*3*512)*512 = 2,359,296
CONV3-512: [28x28x512]  memory:  28*28*512=400K   weights: (3*3*512)*512 = 2,359,296
POOL2: [14x14x512]  memory:  14*14*512=100K   weights: 0
CONV3-512: [14x14x512]  memory:  14*14*512=100K   weights: (3*3*512)*512 = 2,359,296
CONV3-512: [14x14x512]  memory:  14*14*512=100K   weights: (3*3*512)*512 = 2,359,296
CONV3-512: [14x14x512]  memory:  14*14*512=100K   weights: (3*3*512)*512 = 2,359,296
POOL2: [7x7x512]  memory:  7*7*512=25K  weights: 0
FC: [1x1x4096]  memory:  4096  weights: 7*7*512*4096 = 102,760,448
FC: [1x1x4096]  memory:  4096  weights: 4096*4096 = 16,777,216
FC: [1x1x1000]  memory:  1000 weights: 4096*1000 = 4,096,000

TOTAL memory: 24M * 4 bytes ~= 93MB / image (only forward! ~*2 for bwd)
TOTAL params: 138M parameters
~~~

As is common with Convolutional Networks, notice that most of the memory is used in the early CONV layers, and that most of the parameters are in the last FC layers. In this particular case, the first FC layer contains 100M weights, out of a total of 140M.


<a name='comp'></a>

#### Computational Considerations

The largest bottleneck to be aware of when constructing ConvNet architectures is the memory bottleneck. Many modern GPUs have a limit of 3/4/6GB memory, with the best GPUs having about 12GB of memory. There are three major sources of memory to keep track of:

- From the intermediate volume sizes: These are the raw number of **activations** at every layer of the ConvNet, and also their gradients (of equal size). Usually, most of the activations are on the earlier layers of a ConvNet (i.e. first Conv Layers). These are kept around because they are needed for backpropagation, but a clever implementation that runs a ConvNet only at test time could in principle reduce this by a huge amount, by only storing the current activations at any layer and discarding the previous activations on layers below.
- From the parameter sizes: These are the numbers that hold the network **parameters**, their gradients during backpropagation, and commonly also a step cache if the optimization is using momentum, Adagrad, or RMSProp. Therefore, the memory to store the parameter vector alone must usually be multiplied by a factor of at least 3 or so.
- Every ConvNet implementation has to maintain **miscellaneous** memory, such as the image data batches, perhaps their augmented versions, etc.

Once you have a rough estimate of the total number of values (for activations, gradients, and misc), the number should be converted to size in GB. Take the number of values, multiply by 4 to get the raw number of bytes (since every floating point is 4 bytes, or maybe by 8 for double precision), and then divide by 1024 multiple times to get the amount of memory in KB, MB, and finally GB. If your network doesn't fit, a common heuristic to "make it fit" is to decrease the batch size, since most of the memory is usually consumed by the activations.

### Visualizing and Understanding Convolutional Networks

In the [next section](../understanding-cnn/) of these notes we look at visualizing and understanding Convolutional Neural Networks.

<a name='add'></a>

### Additional Resources

Additional resources related to implementation:

- [DeepLearning.net tutorial](http://deeplearning.net/tutorial/lenet.html) walks through an implementation of a ConvNet in Theano
- [cuda-convnet2](https://code.google.com/p/cuda-convnet2/) by Alex Krizhevsky is a ConvNet implementation that supports multiple GPUs
- [ConvNetJS CIFAR-10 demo](http://cs.stanford.edu/people/karpathy/convnetjs/demo/cifar10.html) allows you to play with ConvNet architectures and see the results and computations in real time, in the browser.
- [Caffe](http://caffe.berkeleyvision.org/), one of the most popular ConvNet libraries.
- [Example Torch 7 ConvNet](https://github.com/nagadomi/kaggle-cifar10-torch7) that achieves 7% error on CIFAR-10 with a single model
- [Ben Graham's Sparse ConvNet](https://www.kaggle.com/c/cifar-10/forums/t/10493/train-you-very-own-deep-convolutional-network/56310) package, which Ben Graham used to great success to achieve less than 4% error on CIFAR-10.
