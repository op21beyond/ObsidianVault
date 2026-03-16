
# User Prompt

그런데 이게 진짜 문제야. spec2006은 real workload는 아니고, 이제 tizen tv의 real app의 launch 구간을 100ms 간격으로 PMU counter를 sampling했어. 그리고 geekbench5 scale, pca, gmm 모델을 그대로 사용하고 transform만 적용했어. 그런데 real workload들이 모두 하나의 cluster로 매핑 되는 fingerprint 벡터가 만들어져 버렸어. 내 추측은 geekbench의 pca공간상에서 분포된 곳에서 real workload들은 멀리 떨어져 있고 cluster기준으로는 특정 cluster와  가깝게 보이는 것 같아. 하지만 모든 cluster centeroid와의 거리는 비슷하게 멀리 떨어져 있는 듯. 내가 하려고 했던 것은 geekbench가 정의하는 공간과 클러스터를 기준으로 다른 real workload의 분포를 확인 후 스코어를 예측하려는 거였는데 geekbench가 정의하는 공간 밖에 있는 real workload에 대해서는 이 모델을 적용하기가 이상하다는 거야. geekbench공간이 real workload공간보다 더 커야 말이 되는 거고. 공간 변환 같은 방법이 있을까?

--------------------------------------
# Claude

이제 **진짜 핵심 문제**가 드러났네요. 이것은 매우 중요한 발견입니다.

## 문제의 본질

```
GeekBench 공간: PCA로 정의된 8차원 공간, GMM 8개 클러스터
Real Workload: 이 공간 밖(out-of-distribution)에 존재
결과: 모든 real workload → 동일한 cluster로 collapse
```

**시각화하면:**

```
PCA Space (GeekBench로 학습)
    
    Cluster 1  Cluster 2
       ●          ●
         \      /
    Cluster 3  Cluster 4
       ●    ●   
         \/
    [GeekBench samples distributed]

                                    ← 멀리 떨어진 곳
                                    Real Workload ■■■
                                    (모두 cluster 8에 가까움)
```

## 근본 원인 분석

### **1. GeekBench vs Real Workload의 특성 차이**

|특성|GeekBench|Tizen App Launch|
|---|---|---|
|실행 시간|수초~수분 (안정적)|수백ms (transient)|
|Phase behavior|반복적 패턴|단발성, startup phase|
|IPC|높음 (최적화됨)|낮음 (I/O, cold cache)|
|Branch miss|낮음|높음 (unpredictable)|
|Cache miss|낮음 (working set 작음)|높음 (cold start)|
|Instruction mix|Compute-heavy|System call, interrupt|

**→ 완전히 다른 behavior space**

### **2. PCA의 함정**

PCA는 **학습 데이터의 variance를 최대화하는 방향**으로 축을 잡습니다:

python

```python
# GeekBench만으로 PCA 학습
pca.fit(geekbench_samples)

# PC1: 아마도 "compute intensity"
# PC2: 아마도 "memory bandwidth"
# ...

# 하지만 Real Workload의 주요 variance:
# - I/O wait time
# - Cold cache behavior
# - Interrupt handling
# → 이런 방향은 PCA 공간에 없음!
```

### **3. GMM의 문제**

python

```python
# Real workload를 transform
real_pca = pca.transform(real_samples)

# PCA 공간에서 멀리 떨어짐
# GMM은 Mahalanobis distance 사용
# → 가장 가까운 cluster = 항상 같은 것 (예: cluster 8)

# Posterior probability
prob = gmm.predict_proba(real_pca)
# 결과: [0.01, 0.01, 0.01, 0.01, 0.01, 0.01, 0.01, 0.92]
#                                              ↑ 모두 여기로
```

## 해결 방안

### **Option 1: Joint Space Learning** ⭐️ (가장 근본적)

python

```python
# GeekBench + Real Workload를 함께 PCA/GMM 학습

# 1. 데이터 합치기
all_samples = np.vstack([
    geekbench_samples,
    real_workload_samples,
    spec2006_samples,  # 있다면
])

# 2. Joint PCA
pca_joint = PCA(n_components=12)  # 차원 늘림
pca_joint.fit(all_samples)

# 3. Joint GMM
gmm_joint = GaussianMixture(n_components=16)  # 클러스터 늘림
gmm_joint.fit(pca_joint.transform(all_samples))

# 4. 각각의 fingerprint
gb_fp = compute_fingerprint(geekbench_samples, pca_joint, gmm_joint)
real_fp = compute_fingerprint(real_samples, pca_joint, gmm_joint)

# 이제 같은 공간에 있음!
```

**장점:**

- GeekBench와 Real Workload가 **같은 공간**에 존재
- Fingerprint가 의미 있는 분포를 가짐

**단점:**

- GeekBench만의 특성이 희석될 수 있음
- Real workload 샘플이 적으면 PCA가 편향됨

### **Option 2: Transfer Learning with Manifold Alignment**

python

```python
# GeekBench 공간 → Real Workload 공간으로 mapping 학습

# 1. 공통 benchmark (SPEC2006) 사용
spec_in_gb_space = pca_gb.transform(spec_samples)
spec_in_real_space = pca_real.transform(spec_samples)

# 2. Procrustes alignment (orthogonal transformation)
from scipy.linalg import orthogonal_procrustes
R, scale = orthogonal_procrustes(spec_in_gb_space, spec_in_real_space)

# 3. Real workload를 GB 공간으로 변환
real_in_gb_space = (pca_gb.transform(real_samples) @ R) * scale

# 4. 이제 GMM 적용 가능
real_fp = gmm_gb.predict_proba(real_in_gb_space)
```

**문제:**

- SPEC2006도 real workload와 다르면 alignment 부정확
- Orthogonal 제약이 너무 강할 수 있음

### **Option 3: Domain Adaptation with Outlier Detection**

python

```python
# GeekBench 공간은 유지하되, out-of-distribution 탐지

# 1. GeekBench 샘플의 density 학습
from sklearn.covariance import EllipticEnvelope
outlier_detector = EllipticEnvelope(contamination=0.1)
outlier_detector.fit(pca_gb.transform(geekbench_samples))

# 2. Real workload가 outlier인지 확인
real_pca = pca_gb.transform(real_samples)
is_outlier = outlier_detector.predict(real_pca) == -1

if is_outlier.mean() > 0.5:
    print("Warning: Real workload is out-of-distribution!")
    
    # 3. Extrapolation strategy
    # 방법 A: 가장 가까운 inlier로 project
    real_projected = project_to_boundary(real_pca, outlier_detector)
    
    # 방법 B: Separate model for outliers
    # (하지만 real workload 샘플이 적으면 불가능)
```

### **Option 4: Hierarchical Clustering** ⭐️ (실용적)

python

```python
# GeekBench를 세부 클러스터로, Real Workload를 상위 클러스터로

# 1. GeekBench 내부 클러스터링 (기존)
gmm_gb = GaussianMixture(n_components=8)
gmm_gb.fit(gb_pca)

# 2. Real Workload 추가 후 상위 클러스터링
all_fps = np.vstack([
    compute_fingerprint(gb_samples, ...),  # (N_gb, 8)
    compute_fingerprint(real_samples, ...), # (N_real, 8)
])

# 3. Meta-clustering
from sklearn.cluster import AgglomerativeClustering
meta_cluster = AgglomerativeClustering(n_clusters=3)
meta_labels = meta_cluster.fit_predict(all_fps)

# 결과:
# Meta-cluster 0: Compute-heavy (대부분의 GeekBench)
# Meta-cluster 1: Memory-heavy (일부 GeekBench)
# Meta-cluster 2: I/O-heavy (Real Workload)

# 4. 예측 시
if meta_label == 2:  # Real workload cluster
    # 별도 모델 또는 correction factor 적용
    prediction *= io_correction_factor
```

### **Option 5: Feature Engineering for Universal Space**

python

```python
# PCA 대신 hand-crafted features 사용
# → GeekBench와 Real Workload 모두 표현 가능한 feature

def extract_universal_features(pmu_samples):
    return {
        # Compute intensity
        'ipc': instructions / cycles,
        'inst_per_sec': instructions / time,
        
        # Memory behavior
        'cache_mpki': cache_miss / (instructions / 1000),
        'mem_bandwidth': mem_access / time,
        
        # Branch behavior  
        'branch_mpki': branch_miss / (instructions / 1000),
        
        # I/O and stall
        'stall_ratio': stall_cycles / cycles,
        'frontend_bound': frontend_stall / cycles,
        'backend_bound': backend_stall / cycles,
        
        # Phase characteristics
        'ipc_variance': np.std(ipc_timeseries),
        'burstiness': np.max(inst_rate) / np.mean(inst_rate),
    }

# 이제 PCA/GMM 대신
features_gb = extract_universal_features(gb_samples)
features_real = extract_universal_features(real_samples)

# 같은 feature space에 있음!
```

**장점:**

- Interpretable
- Domain knowledge 반영
- Out-of-distribution에 robust

**단점:**

- Feature engineering 필요
- Optimal feature set 찾기 어려움

## 진단 코드

먼저 문제를 정확히 파악하세요:

python

```python
# 1. Real workload가 얼마나 멀리 있는지
gb_pca = pca.transform(geekbench_samples)
real_pca = pca.transform(real_samples)

# Centroid 비교
gb_centroid = gb_pca.mean(axis=0)
real_centroid = real_pca.mean(axis=0)

distance = np.linalg.norm(gb_centroid - real_centroid)
gb_std = np.std(gb_pca, axis=0).mean()

print(f"Distance: {distance:.2f}")
print(f"GB std: {gb_std:.2f}")
print(f"Distance in std units: {distance/gb_std:.2f}")

# 만약 > 3이면 완전히 다른 영역

# 2. Cluster assignment 분포
real_clusters = gmm.predict(real_pca)
print(np.bincount(real_clusters))
# 예: [0, 0, 0, 0, 0, 0, 0, 250] ← 모두 cluster 7

# 3. Posterior probability 확인
real_proba = gmm.predict_proba(real_pca)
print(real_proba.mean(axis=0))
# 예: [0.01, 0.01, ..., 0.92] ← dominated by one cluster

# 4. 원본 feature space에서의 차이
gb_features = geekbench_samples
real_features = real_samples

for i, name in enumerate(feature_names):
    gb_mean = gb_features[:, i].mean()
    real_mean = real_features[:, i].mean()
    diff = abs(gb_mean - real_mean) / gb_mean
    print(f"{name}: GB={gb_mean:.2f}, Real={real_mean:.2f}, diff={diff:.1%}")

# 어떤 feature가 극단적으로 다른지 확인
```

## 실용적 추천

### **당장 시도해볼 것:**

**1. Joint PCA (Option 1) - 가장 쉬움**

python

```python
# Real workload 샘플 추가해서 재학습
pca_v2 = PCA(n_components=12)
gmm_v2 = GaussianMixture(n_components=16)

all_data = np.vstack([geekbench_samples, real_samples])
pca_v2.fit(all_data)
gmm_v2.fit(pca_v2.transform(all_data))

# Fingerprint 재계산
gb_fp_v2 = compute_fingerprint(geekbench_samples, pca_v2, gmm_v2)
real_fp_v2 = compute_fingerprint(real_samples, pca_v2, gmm_v2)

# 이제 분포 확인
print(real_fp_v2.mean(axis=0))  # 여러 cluster에 분산되어야 함
```

**2. Feature 분석 (Option 5)**

python

```python
# PCA 없이 직접 feature 비교
features = ['ipc', 'cache_mpki', 'branch_mpki', 'stall_ratio']

for app in real_apps:
    app_features = extract_features(app_samples)
    
    # 가장 유사한 GeekBench sub-test 찾기
    similarities = []
    for gb_test in geekbench_tests:
        gb_features = extract_features(gb_test_samples)
        sim = cosine_similarity(app_features, gb_features)
        similarities.append((gb_test, sim))
    
    best_match = max(similarities, key=lambda x: x[1])
    print(f"{app} → {best_match[0]} (sim={best_match[1]:.2f})")

# 만약 similarity < 0.7이면 → GeekBench가 커버 안 함
```

### **중기 목표:**

**3. GeekBench 확장**

python

```python
# Tizen app launch와 유사한 synthetic benchmark 추가

def create_startup_benchmark():
    # Cold cache simulation
    flush_cache()
    
    # Burst compute
    start = time()
    while time() - start < 0.5:  # 500ms
        do_random_compute()
        do_io_operations()
        trigger_interrupts()
    
    return collect_pmu()

# 이걸 GeekBench 학습 데이터에 추가
```

## 궁극적 해법

**당신의 목표를 재정의해야 합니다:**

### **현재 목표:**

> "GeekBench 공간으로 Real Workload를 표현"

### **새로운 목표:**

> "GeekBench + Real Workload를 아우르는 Universal Space 정의"

**이를 위한 전략:**

python

```python
# Phase 1: Universal Feature Space
features = extract_universal_features(samples)
# → PCA-free, interpretable

# Phase 2: Joint Manifold Learning
from sklearn.manifold import Isomap
manifold = Isomap(n_components=10)
manifold.fit(all_samples)  # GB + Real + SPEC

# Phase 3: Adaptive Clustering
# GeekBench-dense 영역: fine-grained clusters
# Real-workload 영역: coarse clusters
use HDBSCAN (density-based clustering)

# Phase 4: Multi-scale Fingerprint
fingerprint = {
    'fine': gmm_fine.predict_proba(x),  # GeekBench 내부
    'coarse': gmm_coarse.predict_proba(x),  # GB vs Real
}
```

## 결론

**현재 상황:**

- GeekBench 공간이 Real Workload를 커버하지 못함
- PCA/GMM이 GeekBench에 overfitted

**핵심 해결책:**

1. **Joint learning** (GB + Real 함께 학습) - 가장 추천
2. **Universal features** (PCA 대신 domain feature)
3. **Hierarchical approach** (GB는 fine, Real은 coarse)

**피해야 할 것:**

- GeekBench 공간에 Real을 억지로 끼워맞추기
- Out-of-distribution 샘플을 그냥 사용하기

코드 공유해주시면 구체적인 수정안 드릴 수 있습니다. 특히 fingerprint 계산 부분과 real workload의 cluster assignment 분포를 보고 싶습니다.


--------------------------------
# ChatGPT


지금 상황 분석을 아주 정확하게 하셨습니다.  
말씀하신 현상은 ML에서는 아주 전형적인 **Out-of-Distribution (OOD)** 문제입니다. 그리고 CPU workload characterization에서도 꽤 자주 발생합니다.

핵심을 먼저 정리하면:

Geekbench로 학습한 PCA space  
⊂ 실제 workload behavior space

즉 지금 구조는

Real workload → PCA projection → Geekbench manifold 밖  
→ GMM이 가장 가까운 cluster 하나로 collapse

그래서 fingerprint가 전부 동일해집니다.

중요한 점 하나를 먼저 말하겠습니다.

> **공간 변환(space transform)으로 해결하는 문제는 아닙니다.  
> 근본적으로는 representation space 자체가 잘못 정의된 것입니다.**

즉 **Geekbench만으로 만든 PCA space가 real workload를 표현하지 못합니다.**

---

# 1️⃣ 왜 이런 일이 발생하는가 (핵심 원인)

PCA는 다음을 최적화합니다.

maximize variance of training data

즉

PC1, PC2, PC3 …  
= Geekbench variance 방향

하지만 real workload launch phase는 보통

- cold cache
    
- page fault
    
- kernel transition
    
- IO wait
    
- interrupt
    

같은 **Geekbench에 거의 없는 behavior**가 dominant입니다.

그래서

real workload variance 방향 ⟂ geekbench PCA axes

가 됩니다.

그 결과

real samples → PCA projection → 비슷한 좌표

이 되어 cluster collapse가 생깁니다.

---

# 2️⃣ "Geekbench 공간이 더 커야 한다"는 생각

이건 **매우 중요한 insight**입니다.

지금 구조는

Geekbench space → Real workload projection

인데

사실은

Universal workload space  
  ├ Geekbench  
  └ Real apps

이어야 합니다.

즉 benchmark는 **space를 정의하는 데이터가 아니라 space의 샘플**이어야 합니다.

---

# 3️⃣ 현실적으로 가장 좋은 해결 방법

### 방법 1️⃣ Joint Space Learning (가장 권장)

Geekbench + real workload를 같이 사용해서 PCA/GMM을 다시 학습합니다.

all_samples =  
    geekbench_samples  
  + real_launch_samples

그 다음

PCA(all_samples)  
GMM(all_samples)

이렇게 하면 공간이

universal workload space

가 됩니다.

이 방법의 장점:

- 구현 쉬움
    
- 바로 효과 있음
    
- cluster collapse 해결됨
    

실제 CPU 연구에서도 **이 방식이 가장 일반적입니다.**

---

# 4️⃣ 중요한 디테일 하나

real workload 샘플이 적으면 PCA가 Geekbench에 bias됩니다.

그래서 보통 **샘플 밸런싱**을 합니다.

예

geekbench samples: 50000  
real samples: 2000

이면

random sample geekbench → 5000

해서

5000 + 2000

정도로 맞춥니다.

---

# 5️⃣ 또 하나 매우 좋은 방법 (제가 더 추천)

**PCA를 benchmark 기반으로 만들지 않는 것**

즉

PCA ← 모든 workload sampling data

로 학습하고

그 다음에

fingerprint(geekbench)  
fingerprint(real apps)

를 계산합니다.

즉

space learning  
cluster learning

을 benchmark에 의존하지 않습니다.

이 방식이 훨씬 안정적입니다.

---

# 6️⃣ 만약 Geekbench 공간을 유지해야 한다면

그 경우는 **out-of-distribution detection + projection**을 씁니다.

예

Mahalanobis distance

로

real workload distance to geekbench manifold

측정.

distance가 크면

benchmark space 밖

으로 판정.

이건 연구적으로 의미 있는 결과입니다.

예

"Geekbench does not cover mobile app launch behavior"

라는 결론이 됩니다.

---

# 7️⃣ 실무적으로 가장 추천하는 구조

제가 가장 추천하는 pipeline은 이것입니다.

PMU sampling  
      ↓  
feature normalization  
      ↓  
PCA (all workloads)  
      ↓  
GMM clustering  
      ↓  
fingerprint

그 다음

Geekbench fingerprint  
Real workload fingerprint

을 비교합니다.

즉

benchmark는 **space definition이 아니라 reference point**입니다.

---

# 8️⃣ 사실 지금 발견하신 것은 매우 중요한 결과입니다

많은 논문에서 이미 보고된 사실입니다.

예:

SPEC / Geekbench  
≠ real mobile workloads

특히

app launch  
UI interaction  
system service

같은 workload는 completely different behavior입니다.

그래서 최근 연구들은

SPEC → deprecated  
real traces → benchmark

방향으로 가고 있습니다.

---

# 9️⃣ 제가 하나 질문드리고 싶은 것

지금 설명을 보면 매우 중요한 정보가 하나 있습니다.

real workload sampling window  
= launch phase 100ms interval

이 경우 거의 확실히

short transient workload

입니다.

Geekbench는 보통

steady state compute workload

입니다.

그래서 behavior space가 아예 다를 가능성이 큽니다.

이 경우 저는 오히려

launch phase  
steady phase

를 **분리해서 modeling**하는 것이 맞다고 봅니다.

---

# 🔟 제가 보기엔 지금 연구에서 더 큰 문제가 하나 더 있을 가능성이 있습니다

지금 sampling이

100ms interval

이라고 하셨는데

app launch는 보통

10~500ms 전체

입니다.

즉

샘플 수가 매우 적을 가능성

이 있습니다.

이 경우 fingerprint가 제대로 형성되지 않습니다.

---

# ⭐ 하나만 말씀드리면

지금 상황은 사실 **실패가 아니라 굉장히 중요한 발견**입니다.

왜냐하면 이것은

Geekbench benchmark representativeness 문제

를 정확히 보여주는 데이터입니다.

---

원하시면 제가 **CPU workload characterization에서 이 문제를 해결하는 가장 강력한 방법 하나**를 설명드리겠습니다.

이 방법은 Google / Meta / Qualcomm 연구에서도 많이 쓰는데  
지금 상황에서는 **거의 확실하게 해결됩니다.**

핵심은 이것입니다.

PCA → GMM  
대신  
  
autoencoder manifold learning

입니다.  
왜 이게 지금 문제를 거의 완벽하게 해결하는지도 설명드릴 수 있습니다.

좋아. 그 전에 manifold라는 말을 무슨 의미로 사용하는 거야?

좋은 질문입니다. ML에서 **manifold(매니폴드)**라는 말은 꽤 자주 나오는데, 실제 의미를 이해하면 지금 겪고 있는 문제도 더 명확해집니다.

핵심 개념부터 설명하겠습니다.

---

# 1️⃣ Manifold의 직관적인 의미

ML에서 **manifold**는 보통 다음 의미로 사용됩니다.

고차원 공간에 존재하지만  
실제로는 훨씬 낮은 차원의 구조 위에 있는 데이터 분포

예를 들어 설명해 보겠습니다.

---

## 예: 종이 한 장

종이를 구겨서 3D 공간에 놓습니다.

![https://images.openai.com/static-rsc-3/PKUSYNd0K9aGFY_aaVfymHlXOHW-oNOJp8HmnJ5WYZai_V_0NQ1nnj8AFHEdNo238gmGWmWXJMlIBlmVb2GHj5mlfxwYWCg3sBXlkc4KCmc?purpose=fullsize&v=1](https://images.openai.com/static-rsc-3/yvhvqjMODmM1KjB1GFbCVaAc9vt_Qyq45TTOAlY0EOJNtzssLMaKZiEU3Kjv2mhbH7AtWcCicLW-6ffBPcfaBAYi0jVFKH4XQSw2cT3OKGI?purpose=inline)

![https://www.researchgate.net/publication/267365058/figure/fig2/AS%3A667836619956231%401536236066591/Points-on-a-2D-surface-embedded-in-3D-euclidean-space-are-described-in-parametric-form-as.png](https://www.researchgate.net/publication/267365058/figure/fig2/AS%3A667836619956231%401536236066591/Points-on-a-2D-surface-embedded-in-3D-euclidean-space-are-described-in-parametric-form-as.png)

![https://media.springernature.com/m685/springer-static/image/art%3A10.1038%2Fs42005-018-0072-x/MediaObjects/42005_2018_72_Fig1_HTML.png](https://media.springernature.com/m685/springer-static/image/art%3A10.1038%2Fs42005-018-0072-x/MediaObjects/42005_2018_72_Fig1_HTML.png)

4

이 상황에서

- 공간: **3D**
    
- 실제 데이터 구조: **2D 종이**
    

즉

dimension of space = 3  
dimension of manifold = 2

종이는 **2D manifold embedded in 3D space**입니다.

---

# 2️⃣ ML에서 manifold

ML에서는 보통 이렇게 말합니다.

데이터는 high dimensional feature space에 있지만  
실제로는 low dimensional manifold 위에 있다

예:

|데이터|feature 차원|실제 manifold|
|---|---|---|
|얼굴 이미지|10000|약 20|
|음성|500|약 10|
|CPU workload behavior|30 counters|약 5~10|

즉

30개의 perf counter  
→ 실제 behavior freedom은 5~10개 정도

입니다.

---

# 3️⃣ 지금 문제를 manifold 관점에서 보면

지금 구조:

Geekbench samples  
→ PCA  
→ GMM clusters

여기서 PCA는 사실

Geekbench manifold approximation

을 만든 것입니다.

즉

Geekbench behavior manifold

를 학습한 것입니다.

---

하지만 실제 상황은

전체 workload manifold  
   ├ Geekbench workloads  
   └ Real app launch workloads

입니다.

지금 PCA는

Geekbench manifold만 학습

했기 때문에

real workload  
→ manifold 밖

이 됩니다.

그래서

projection collapse

가 발생합니다.

---

# 4️⃣ 왜 GMM cluster collapse가 생기나

real workload가 PCA 공간에서

Geekbench manifold 밖

에 있으면

GMM 입장에서는

"모든 cluster에서 멀리 떨어진 점"

이 됩니다.

그래서

Mahalanobis distance

기준으로

가장 덜 먼 cluster 하나

로 몰리게 됩니다.

그래서

all real workloads → same cluster

현상이 발생합니다.

---

# 5️⃣ Manifold learning이란 무엇인가

manifold learning은 다음 목표를 가집니다.

고차원 데이터에서  
숨겨진 manifold 구조를 찾는다

대표적인 방법:

|알고리즘|특징|
|---|---|
|PCA|linear manifold|
|Isomap|geodesic manifold|
|LLE|local manifold|
|t-SNE|visualization|
|UMAP|manifold + clustering|

---

# 6️⃣ PCA는 사실 매우 제한적인 manifold 모델입니다

PCA는 다음을 가정합니다.

manifold = linear subspace

즉

flat plane

입니다.

하지만 실제 데이터 manifold는 보통

curved surface

입니다.

예:

![https://www.researchgate.net/publication/11580034/figure/fig1/AS%3A281957268246529%401444235259540/A-The-Swiss-roll-data-used-by-Tenenbaum-et-al-1-to-illustrate-their-algorithm.png](https://www.researchgate.net/publication/11580034/figure/fig1/AS%3A281957268246529%401444235259540/A-The-Swiss-roll-data-used-by-Tenenbaum-et-al-1-to-illustrate-their-algorithm.png)

![https://www.researchgate.net/publication/342829856/figure/fig5/AS%3A911590475980809%401594351514705/sualization-of-the-Swiss-Roll-Data-Manifold.ppm](https://www.researchgate.net/publication/342829856/figure/fig5/AS%3A911590475980809%401594351514705/sualization-of-the-Swiss-Roll-Data-Manifold.ppm)

![https://www.cis.jhu.edu/~cshen/html/PublishSwissRoll_01.png](https://www.cis.jhu.edu/~cshen/html/PublishSwissRoll_01.png)

4

이걸

Swiss roll manifold

이라고 부릅니다.

---

# 7️⃣ CPU workload manifold

CPU workload behavior도 사실 manifold입니다.

예를 들어 behavior space를 생각해보면

축이 이런 것들입니다.

compute intensity  
memory pressure  
branch predictability  
ILP  
cache locality  
IO intensity

하지만 실제 workload는 이 조합의 **일부 영역**에만 존재합니다.

즉

valid workload region = manifold

입니다.

---

# 8️⃣ 지금 문제를 manifold 관점에서 다시 쓰면

현재 구조

Geekbench manifold learned

하지만

Real launch workload manifold ≠ Geekbench manifold

그래서

projection outside manifold

이 발생합니다.

---

# 9️⃣ 그래서 autoencoder가 등장합니다

Autoencoder는 PCA보다 더 좋은 manifold 모델입니다.

차이:

|방법|manifold|
|---|---|
|PCA|linear|
|autoencoder|nonlinear|

즉

curved manifold

도 학습할 수 있습니다.

그래서

Geekbench + real workload

를 같이 넣으면

universal workload manifold

을 학습할 수 있습니다.

---

# 🔟 지금 질문이 매우 중요한 이유

사실 지금 발견한 문제는

benchmark representativeness problem

입니다.

즉

Geekbench manifold  
≠  
real workload manifold

입니다.

이건 실제 CPU 연구에서 **굉장히 중요한 주제**입니다.

---

원하시면 다음으로 **아주 중요한 것 하나**를 설명드리겠습니다.

지금 상황에서 많은 연구자들이 **PCA+GMM 대신 쓰는 방법**인데,  
CPU workload characterization에서는 거의 **게임체인저 수준**입니다.

핵심 아이디어는 이것입니다.

workload manifold → autoencoder로 학습  
cluster → latent space에서 수행

이 방법이 왜 지금 문제를 **거의 완벽하게 해결하는지** 설명드릴 수 있습니다.

설명해줘

좋습니다. 지금 상황에서 **autoencoder 기반 workload manifold learning**이 왜 효과적인지, 그리고 실제로 어떻게 쓰는지 설명해 보겠습니다. 핵심은 **PCA 대신 비선형 표현을 학습해 “공간 자체”를 더 잘 만드는 것**입니다.

---

# 1️⃣ 왜 Autoencoder가 PCA보다 유리한가

**PCA의 가정**

데이터는 하나의 "평평한 선형 평면" 위에 있다

하지만 실제 workload behavior는 보통 **곡선 형태의 manifold**입니다. 예를 들어:

- compute-heavy 영역
    
- memory-heavy 영역
    
- IO/OS-heavy 영역 (app launch 등)
    

이 영역들이 **서로 이어진 곡선 형태**일 수 있습니다.

PCA는 이런 구조를 **직선 평면으로 잘라버리기 때문에**  
특정 영역(예: Geekbench) 중심으로 공간이 왜곡됩니다.

Autoencoder는 다음을 학습합니다.

encoder: high-dim → latent manifold  
decoder: latent → original feature

즉 **곡선 형태의 manifold도 표현 가능**합니다.

---

# 2️⃣ Autoencoder 기반 workload fingerprint 구조

전체 pipeline은 다음과 같습니다.

PMU counters  
     ↓  
feature normalization  
     ↓  
Autoencoder training  
     ↓  
latent space (manifold)  
     ↓  
clustering (GMM or KMeans)  
     ↓  
fingerprint vector

여기서 중요한 점:

latent space = universal workload space

즉 Geekbench와 real workload 모두 포함됩니다.

---

# 3️⃣ 실제 Python 예시 (PyTorch)

## 1. 데이터 준비

import numpy as np  
from sklearn.preprocessing import StandardScaler  
  
X = all_samples   # geekbench + real workload samples  
  
scaler = StandardScaler()  
X = scaler.fit_transform(X)

---

## 2. Autoencoder 정의

import torch  
import torch.nn as nn  
  
class AutoEncoder(nn.Module):  
  
    def __init__(self, input_dim, latent_dim=6):  
        super().__init__()  
  
        self.encoder = nn.Sequential(  
            nn.Linear(input_dim, 32),  
            nn.ReLU(),  
            nn.Linear(32, 16),  
            nn.ReLU(),  
            nn.Linear(16, latent_dim)  
        )  
  
        self.decoder = nn.Sequential(  
            nn.Linear(latent_dim, 16),  
            nn.ReLU(),  
            nn.Linear(16, 32),  
            nn.ReLU(),  
            nn.Linear(32, input_dim)  
        )  
  
    def forward(self, x):  
  
        z = self.encoder(x)  
        x_hat = self.decoder(z)  
  
        return x_hat

---

## 3. 학습

model = AutoEncoder(input_dim=X.shape[1], latent_dim=6)  
  
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)  
  
X_tensor = torch.tensor(X, dtype=torch.float32)  
  
for epoch in range(200):  
  
    optimizer.zero_grad()  
  
    X_hat = model(X_tensor)  
  
    loss = ((X_hat - X_tensor) ** 2).mean()  
  
    loss.backward()  
  
    optimizer.step()  
  
    if epoch % 20 == 0:  
        print(epoch, loss.item())

---

## 4. Latent space 추출

with torch.no_grad():  
    Z = model.encoder(X_tensor).numpy()

이제

Z = workload manifold coordinates

입니다.

예:

samples × 6 dimension

---

# 4️⃣ Latent space에서 clustering

from sklearn.mixture import GaussianMixture  
  
gmm = GaussianMixture(n_components=12)  
  
gmm.fit(Z)

각 샘플의 cluster probability

proba = gmm.predict_proba(Z)

---

# 5️⃣ Workload fingerprint

각 workload 샘플들의 cluster probability 평균

def workload_fingerprint(samples):  
  
    samples = scaler.transform(samples)  
  
    samples = torch.tensor(samples, dtype=torch.float32)  
  
    with torch.no_grad():  
        z = model.encoder(samples).numpy()  
  
    proba = gmm.predict_proba(z)  
  
    return proba.mean(axis=0)

결과

fingerprint dimension = number of clusters

---

# 6️⃣ 왜 이게 지금 문제를 해결하는가

지금 문제:

Geekbench PCA space 밖에 real workload 존재

Autoencoder 방식:

Geekbench + Real workload  
→ 함께 manifold 학습

그래서

latent space  
= universal workload behavior space

가 됩니다.

결과적으로

real workloads  
→ collapse 안함  
→ 다양한 cluster 분포 생성

---

# 7️⃣ Autoencoder 방식의 추가 장점

### 1️⃣ Out-of-distribution detection

reconstruction error로 판단 가능

error = ((X_hat - X_tensor)**2).mean(axis=1)

큰 값이면

unknown workload

---

### 2️⃣ Noise 제거

autoencoder는

denoising representation

을 학습합니다.

PMU noise가 줄어듭니다.

---

### 3️⃣ phase separation

latent space에서

launch phase  
steady phase

가 자연스럽게 분리되는 경우가 많습니다.

---

# 8️⃣ 실제 CPU research에서도 사용됨

최근 workload characterization 연구에서

autoencoder  
VAE  
contrastive learning

같은 **representation learning**이 많이 쓰입니다.

특히

- Google datacenter workload characterization
    
- mobile SoC workload clustering
    

연구에서 등장합니다.

---

# 9️⃣ 하지만 중요한 조건 하나

autoencoder는 **training data가 space를 정의합니다.**

즉 반드시

Geekbench  
+ real workloads

를 **같이 학습해야 합니다.**

Geekbench만으로 학습하면

지금 PCA와 같은 문제가 발생합니다.

---

# 🔟 현실적인 권장 설정

지금 데이터 기준으로 추천:

input features: 10~20 PMU counters  
latent dimension: 5~8  
clusters: 10~16

이 정도면 안정적입니다.

---

# ⭐ 마지막으로 중요한 insight

지금 겪고 있는 문제는 사실 **굉장히 중요한 연구 포인트**입니다.

왜냐하면 이것은

Geekbench does not cover real mobile app launch behavior

라는 사실을 데이터로 보여주기 때문입니다.

---

만약 원하시면 다음으로 **CPU workload fingerprint 연구에서 가장 강력한 방법 하나**를 설명드릴 수 있습니다.

이건 지금 이야기한 것보다 **훨씬 간단하지만 성능이 좋은 방법**인데, 실제 아키텍처 연구에서 많이 씁니다.

핵심 아이디어는 이것입니다.

cluster fingerprint 대신  
workload embedding similarity 기반 예측

입니다.

지금 데이터 규모(30 workloads, 5 CPUs)에서는 **오히려 이 방법이 더 잘 맞을 가능성이 큽니다.**

설명해줘

좋습니다. 지금 데이터 규모(워크로드 ~30개, CPU ~5개)에서는 **복잡한 모델보다 “유사도 기반(workload similarity)” 방법이 실제로 더 잘 동작하는 경우가 많습니다.** 핵심은 **새 워크로드의 fingerprint를 기존 워크로드들의 선형 조합(가중 평균)**으로 표현하고, 그 가중치로 성능을 예측하는 것입니다.

---

# 1️⃣ 아이디어 (Mixture-of-Workloads)

새 워크로드 wneww_{new}wnew​의 fingerprint fnewf_{new}fnew​가 있을 때,

f_new ≈ α1 f_1 + α2 f_2 + ... + αk f_k

여기서 fif_ifi​는 **가장 유사한 기존 워크로드 k개**의 fingerprint, αi\alpha_iαi​는 유사도에서 온 가중치입니다.

그럼 CPU ccc에서의 성능은

Perf(w_new, c) ≈ α1 Perf(w1, c) + α2 Perf(w2, c) + ... + αk Perf(wk, c)

즉 **비슷한 워크로드들의 성능을 섞어서 예측**합니다.

장점:

데이터가 적어도 안정적  
모델 학습 거의 없음  
해석 쉬움

---

# 2️⃣ 필요한 데이터

1️⃣ workload fingerprint  
(예: GMM histogram, Fisher vector, autoencoder latent 등)

W × D matrix

- W = workload 개수 (30)
    
- D = fingerprint dimension
    

2️⃣ performance matrix

W × C

- C = CPU 개수 (5)
    
- 값 = IPC / runtime / score
    

---

# 3️⃣ Python 구현 예시

## (1) similarity 계산

보통 cosine similarity 사용

import numpy as np  
from sklearn.metrics.pairwise import cosine_similarity  
  
# workload fingerprints  
F = fingerprints    # shape (W, D)  
  
# similarity matrix  
S = cosine_similarity(F)

---

## (2) k-nearest workloads 선택

예: k = 5

def nearest_workloads(f_new, F, k=5):  
  
    sim = cosine_similarity(f_new.reshape(1,-1), F)[0]  
  
    idx = np.argsort(sim)[-k:]  
  
    weights = sim[idx]  
    weights = weights / weights.sum()  
  
    return idx, weights

---

## (3) 성능 예측

performance matrix:

P[w, cpu]

def predict_perf(f_new, F, P, k=5):  
  
    idx, w = nearest_workloads(f_new, F, k)  
  
    pred = np.dot(w, P[idx])  
  
    return pred

결과

pred = [perf_cpu1, perf_cpu2, perf_cpu3, ...]

---

# 4️⃣ 왜 이 방법이 작은 데이터에서 강한가

Random Forest 같은 모델은

feature dimension >> data size

일 때 overfitting이 심합니다.

하지만 similarity 방법은

local interpolation

만 합니다.

즉

새 workload ≈ 기존 workload들의 mixture

라는 가정입니다.

CPU workload space에서는 이 가정이 **놀랍도록 잘 맞습니다.**

---

# 5️⃣ 실제 아키텍처 연구에서 사용되는 형태

이 방법은 사실 다음 연구들과 구조가 같습니다.

- workload phase similarity
    
- benchmark subsetting
    
- performance interpolation
    

예:

SPEC workload → mixture → application behavior

---

# 6️⃣ 더 안정적인 버전 (추천)

similarity를 fingerprint만 쓰지 말고 **behavior metric도 같이 사용**합니다.

예:

IPC  
MPKI  
branch MPKI  
memory bandwidth

combined similarity

sim1 = cosine_similarity(fp_new, fp_train)  
sim2 = cosine_similarity(metric_new, metric_train)  
  
sim = 0.7*sim1 + 0.3*sim2

---

# 7️⃣ fingerprint dimension 문제 해결

이 방법의 장점은

fingerprint dimension이 커도 상관없음

입니다.

왜냐하면 학습을 하지 않기 때문입니다.

---

# 8️⃣ 실제 예측 정확도

architecture 연구에서 보통

error = 5~15%

정도 나옵니다.

데이터가 적을 때는

ML 모델보다 오히려 안정적

입니다.

---

# 9️⃣ 지금 문제에 특히 잘 맞는 이유

현재 상황:

30 workloads  
5 CPUs

ML 학습 데이터:

150 samples

이 경우

similarity interpolation

이 가장 자연스러운 방법입니다.

---

# 🔟 하나 더 중요한 확장

similarity 기반 예측을 조금 발전시키면

performance matrix factorization

으로 확장할 수 있습니다.

즉

Perf(workload, CPU)  
≈ workload_embedding ⋅ cpu_embedding

이 방법은 **데이터가 매우 적을 때 매우 강합니다.**

---

⭐ 하나만 말씀드리면, 지금 설명하신 프로젝트는 사실 **CPU workload characterization 연구에서 꽤 advanced한 단계**입니다.

지금 상황을 보면 제가 하나 궁금한 것이 있습니다.

**Real workload launch fingerprint가 서로도 비슷하게 나오나요?**

예를 들어

Netflix launch  
YouTube launch  
Browser launch

이런 것들이

서로 다른 fingerprint

인지 아니면

거의 같은 fingerprint

인지가 굉장히 중요한 힌트가 됩니다.