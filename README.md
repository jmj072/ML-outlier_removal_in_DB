# DB를 활용한 오류 데이터의 제거


## 프로젝트 진행 배경
- 비접촉 Radar로 심박수와 측정하여, 접촉(ECG) 심박수 측정값을 예측하는 모델을 생성
- 즉, Radar로 측정한 데이터만을 이용하여 심박수를 정확하게 예측하고자 함
- 정확한 예측 모델을 생성한 후. 다른 데이터셋에 적용하여 예측값의 오차를 이용해 오류 데이터를 미리 제외하고자 함  
- `협업기업의 요청으로 인해 작업 코드를 공개하기 어렵습니다.`

<br>

## 프로젝트 목표

> ☑ ECG 심박수 데이터가 없어 예측값의 오차를 계산할 수 없더라도, 이상치가 될 가능성이 있는 데이터를 예측하여 미리 제외하고자 함  
> ☑ 오류 데이터를 제외하여 정확한 예측 모델을 생성할 수 있고, 이로 인하여 측정기기의 성능을 높임

<br>

## 개발 환경
-   Python 3.8
-   MySQL
-   Google Colab
-   사용된 library & Tools
    -   `pymysql`, `pandas`,  `numpy`,  ,`scipy`, `pyhrv`, `matplotlib`,  `seaborn`,  `scikit-learn`,  `tensorflow`, `keras`

<br>

## Pipeline
<img src = "https://user-images.githubusercontent.com/77204538/175820393-b3c8818f-449d-4a63-928e-d5d42fbbceca.png" >

<br>

## 1. Dataset
- MySQL 내의 레이더 측정 데이터셋 확인
	- 측정 데이터셋은 `측정날짜_측정대상자_측정거리_측정자세_측정행동.csv` 형태로 구성
		- 측정 자세 (sit,lie, stand)
		- 측정 거리 (0.5m ~ 2m)
		- 측정 당시 행동 (run, active, computer, none)
		- 측정 대상자

- 측정 데이터셋의 평균 오류율 확인
	- 기업측이 제공한 함수를 이용해 각 데이터셋의 평균 오류율 계산
  <img src = "https://user-images.githubusercontent.com/77204538/175820843-9ac7b040-d3d7-40cd-b00c-71f495955013.png"  width=400 height=250>

	- 평균 오류율(check_all BPM error)이 10% 이내인 데이터 확인
	- 측정자세와 측정거리별로 데이터량이 가장 많은 데이터셋 7개를 선정
		- 앉은 자세 측정(측정거리 0.5m, 1m, 1.5m, 2m)
		- 누워있는 자세 측정(측정거리 0.5m, 1m)
		- 서있는 자세 측정(측정거리 1m)
		- *선정되지 않은 측정자세/측정거리 데이터셋은 오류율이 10%를 넘어서므로 제외하였음* 

<br>

## 2. 데이터 전처리
### 2.1 ECG 심박수 데이터 전처리
- ECG 심박수 데이터를 표준화한 후, Z-value > 2인 값은 **이상치**로 판단하여 **ECG 심박수의 평균값**으로 변환
	<img src = "https://user-images.githubusercontent.com/77204538/175821191-0dbd86d6-2971-490d-8a93-608688aaf72a.png"  width=650 height=200>

### 2.2 레이더로 측정된 심박수 데이터 전처리
- 측정된 심박수 데이터를 표준화 한 후, Z-value > 2인 값은 **이상치**로 판단하여 **측정값의 평균**으로 변환
	<img src = "https://user-images.githubusercontent.com/77204538/175821303-090ce312-023c-42d3-a79a-4655ddd6700d.png"  width=650 height=300>

	- Red : 전처리 전 데이터
	- Green : 전처리 후 데이터

<br>


## 3. Modeling
### 3.1 Base 모델 선택
- Random Forest, SVM, XGBoost, ANN Regression 모델 중, 가장 성능이 좋은 모델을 선택 
- 성능 지표는 **오류율의 평균값**을 사용
	- $오류율 = \frac{\left|예측값 - 실측값\right|}{예측값} * 100$
- `Random Forests`모델의 성능이 가장 좋아서 선택함

	|  평균 오류율  	| Dataset1  	| Dataset2 	| Dataset3 	| Dataset4 	| Dataset5 	|    Dataset6   	| Dataset7 	|
	|:-------------:	|:---------:	|:--------:	|:--------:	|:--------:	|:--------:	|:-------------:	|:--------:	|
	| **Random Forest** 	|    **0.19**   	|   **0.27**   	|   **0.24**   	|   **0.24**   	|   **0.30**   	| **1.700000e-01**  	|   **0.26**   	|
	|      SVM      	|    0.62   	|   0.83   	|   0.54   	|   0.45   	|   0.45   	| 3.000000e-01  	|   1.07   	|
	|    XGBoost    	|    0.38   	|   0.32   	|   0.26   	|   0.39   	|   0.39   	|  1.800000e-01 	|   0.30   	|
	|      ANN      	|    0.62   	|   0.91   	|   0.56   	|   0.52   	|   0.52   	|  1.406834e+31 	|   1.17   	|

### 3.2 Hyperparameter tuning
- Rnadomsearch CV로 tuning 진행
	- 모든 데이터셋의 column 개수가 다르기 때문에, 다양한 parameter를 빠르게 분석할 수 있는 방법을 선택함
    - 성능 지표는 **오류율의 평균값**을 사용
    
	|    평가지표    	| Dataset1  	| Dataset2 	| Dataset3 	| Dataset4 	| Dataset5 	| Dataset6 	| Dataset7 	|
	|:--------------:	|:---------:	|:--------:	|:--------:	|:--------:	|:--------:	|:--------:	|:--------:	|
	|       MAE      	|    0.47   	|   0.46   	|  0.26    	|  0.21    	|   0.37   	|  0.21    	|  0.64    	|
	|       MSE      	|    0.58   	|   0.75   	|   0.21   	|   0.08   	|   0.51   	|   0.08   	|   0.90   	|
	|       R2       	|    0.96   	|   0.95   	|   0.97   	|   0.80   	|   0.97   	|   0.97   	|   0.98   	|
	| **Error rate** 	|  **0.52** 	| **0.60** 	| **0.31** 	| **0.33** 	| **0.43** 	| **0.27** 	| **0.74** 	|

<br>

## 4. 오류 데이터 제거 전/후 비교

- 오류율의 히스토그램 상에서, 전체의 95%을 넘어서는 데이터는 기존 데이터셋에서 제외
	- 전체의 95%를 넘어서는 경우는 **오류율이 1~2% 이상의 값**으로, 모델 예측의 오류율이 1% 이내로 유지될 수 있도록 기준을 선정하였음
	
		<img src = "https://user-images.githubusercontent.com/77204538/175822860-a073bec5-222b-4bb3-9813-166d6426867a.png"  width=400 height=400>

	- 각 데이터셋의 validation set 오류율 히스토그램 
	- Line : 전체의 95%를 표시하는기준선

<br>

- 제외 이후, 이전 model을 재학습

	<img src = "https://user-images.githubusercontent.com/77204538/175822989-0dc15340-8ef0-4a87-bc46-f9fb378f154c.png"  width=700 height=200>
	
	- Dataset 1, 2, 3, 4는 미세한 성능 향상을 보였으나,  5, 6, 7은 오히려 예측 성능이 낮아졌음
	
## 5. 결론

- 오류 데이터를 제거한 이후, 미세한 성능 향상을 보였음 
- 예측 모델의 평균 오류율이 1% 미만으로 매우 좋은 성능을 보였음
> ☑ 오류 데이터를 제거한 이후, 예측 모델의 성능 향상을 확인하였음

<br>

## 6. 한계점 및 개선사항

### 한계점
- 오류 데이터 제거 이후의 미세한 성능향상
	- 예측 결과의 오류율 분포를 살펴보았을 때, 모델의 성능 향상이 보였으나, 평가 지표상으로는 미세한 성능향상임.

- 일부 데이터셋에서 오류 데이터 제거 이후, 모델 성능의 감소가 확인
	- 오류 데이터를 제거했음에도, 여전히 오류율 2% 이상의 데이터가 확인됨
		- Holdout 방법으로 데이터셋을 나누었기 때문에, 우연히 validation set에 포함된 오류 데이터만 제거 할 수 있었음 ⇒ training set에 여전히 오류 데이터가 남아있을 가능성이 있음
	- EDA 과정에서 일부 특성들간의 연관관계(상관계수 0.5 이상)가 확인 ⇒ 다중공선성에 의해 회귀모델의 성능이 저하된 것으로 예상됨

<br> 

### 개선사항
- Kfold  교차검증과 같은 전체 데이터셋을 살펴볼 수 있는 방법으로 모델 학습을 할 것
- 연관되어있는 특성들은 PCA로 전처리 한후, 모델 학습을 진행하면 개선될 수 있음

<br> 

## 후기
- 처음 진행했던 팀 프로젝트로, 협업을 통해 문제 해결하는 과정을 경험할 수 있었습니다.
	- 레이더로 측정한 데이터를 전처리 하는 과정에서, 공학적인 내용이 있어 이해하기 어려웠지만, 다른 조원분이 관련 전공을 하셔서 내용을 이해하는데 많은 도움을 받았습니다.
	
- 적극적인 의사소통으로 문제 정의를 명확히 하려 하였습니다.
	- 처음 프로젝트를 진행할 때, 과제 수행 목표가 이해가 되지 않아서 2차례 기업 측에 질문한 적이 있습니다.
		- Training Dataset을 독자적으로 선정하여 하는 것인지(아무 기준을 만들어서), 아니면 기업 측에서 사용하는  기준이 있는 것인지 질문하였습니다. 
		- 이후, 기업 측에서 데이터셋 선정 기준에 대해 추가로 설명을 해주셔서 어떻게 모델을 형성해야하는지 명확히 할 수 있었습니다.
	- 측정 자세/측정시 행동에 대한 설명과 데이터셋 내의 특성들에 대한 설명 없어서, 구체적으로 어떤 의미인지 질문하였습니다.
	- 데이터에 대한 질문을 지속적으로 하여 데이터를 명확히 이해하려 노력하였고, 기업 측에서도 이에대해 긍정적인 피드백을 주셨습니다.

- 오류율이 10%가 넘는 다른 데이터셋을 Test set으로 사용하려 하였으나, 시간 부족으로 진행하지 못했습니다.
	- 같은 조원분이 코로나19 감염으로 도중에 하차하여, 분업하여 진행하려했던 Modeling을 혼자 진행하게 되었습니다.
	- 예측 모델의 성능을 높이기 위한 과정은 진행되었지만, 실제 다른 데이터셋에서 어떤 성능을 보이는지는 확인하지 못하여 아쉬웠습니다.
