## 1. 컴팩션 필요 여부 확인 주기

![image.png](attachment:8ab718e6-f621-4322-8de8-1c085a358249:image.png)

리전서버당 `CompactionChecker` 내부 정적 클래스가 존재한다.

해당 클래스는 `initializeThreads`  에서 생성되며, 스레드 실행 주기도 함께 생성자로 전달된다.

![image.png](attachment:dc0f6924-e177-4b27-98f8-a773ae66776b:image.png)

![image.png](attachment:32be6a59-f2af-4a7e-8bc7-f78daf85fb5e:image.png)

sleepTime 주기로 `CompactionChecker` 클래스의 `run()`이 수행되며

![image.png](attachment:23eea81d-04af-4ae1-a282-6334939a7395:image.png)

sleepTime값으로 전달된 `compationCheckFrequency`는 위 설정을 따라간다.

![image.png](attachment:fdbe6690-4444-4d7b-95d4-3457cdfa90c9:image.png)

`hbase.server.thread.wakefrequency` 기본값이 이미 지정된 상태.

위에서 지정한 sleepTime에 따라, 10초마다 `CompactionChecker`의 `run()` 이 실행되고,

`run()` → `chore()` 를 호출한다.

- `run()` → `chore()` 호출

![image.png](attachment:a35e3a9e-6d8a-40fb-b448-1319047d30c8:image.png)

- `chore()` 내부에서 컴팩션 필요 여부 판단

![image.png](attachment:21ca09cf-cb1a-4085-8e4c-53bd0bc9bb87:image.png)

`multiplier` 값은 `s.getCompactionCheckMultiplier()` 실행 결과이고, 해당 값은 
`hbase.server.compactchecker.interval.multiplier` 설정을 따라간다.

![image.png](attachment:8bf09f08-7e3e-452b-a01f-648b35fb4822:image.png)

`multiplier` 값은 1000이 기본값이다.

if 문 내부에 진입하기 위해서는 `iteration` 값이 `multiplier` 값의 배수여야 한다.

if 문 내부에 진입하지 못한다면 `iteration` 값을 1 증가시키고 `chore()` 가 종료되는데,

위 설정에 따르면 약 2시간 46 주기로 `chore()` 메서드가 1000번 실행되고, 

`iteration` 값이 1000의 배수가 됨에 따라 if 문 내부에 진입한다

→ `CompactionChecker` 클래스의 `chore()`는 10초마다 실행되지만, 실제 CF 단위 compaction 판단은 약 2시간 46분 주기로 수행된다

---

## 2. 컴팩션 필요 여부 확인 및 요청

1. 마이너 컴팩션 여부 확인.
2. 마이너 컴팩션이 필요 없다면, 메이저 컴팩션 주기를 확인한다.

마이너 컴팩션 필요에 따라 컴팩션이 요청된다고 하더라도, 내부적으로 메이저 컴팩션으로 동작할 수 있다.

![image.png](attachment:9bd5d759-03ee-4d9d-bb5c-347da065ec11:image.png)

HStore는 리전 내의 컬럼 패밀리를 뜻한다.

각각의 컬럼 패밀리에 대해 컴팩션 필요 여부를 확인하는데 먼저 `s.needsCompaction()`이 호출된다.

컬럼 패밀리의 컴팩션 필요 여부를 확인한다.

![image.png](attachment:9dbd3319-a6f7-4a7d-b0fb-b74466c9ba37:image.png)

`numCandidates` = 전체 스토어파일 수 - 현재 컴팩션 진행 중이거나, 컴팩션 예정인 파일 수

위 값이 `comConf.getMinFilesToCompact()` 이상이어야 한다.

`getMinFilesToCompact()` 값은 `hbase.hstore.compactionThreshold`  에서 가져온다.

![image.png](attachment:7e61e773-f014-4b3d-8933-6d742abe254a:image.png)

위 설정에 따르면 계산 결과값이 3 이상이면 컴팩션이 필요하다고 판단한다

마이너, 메이저 구분은 이후에 결정된다.

![image.png](attachment:fb81f6d4-a725-4615-8e87-7543d7d2c99c:image.png)

true 반환 시 시스템 컴팩션이 수행된다.

시스템 컴팩션의 우선순위는 `NO_PRIORITY`로써, `Integer.MIN_VALUE` 값으로 적용된다.

![image.png](attachment:c8ef3103-057a-4b54-9af5-3f1cf48a4fe7:image.png)

스토어파일 수가 충분히 줄어 else if 로 빠지는 경우 `shouldPerformMajorCompaction` 메서드가 수행되는데 여기서 메이저 컴팩션 주기를 확인한다.

![image.png](attachment:a02199a2-fe67-4bb5-ae6a-a53392beda04:image.png)

![image.png](attachment:a604a708-fe2a-4d1c-8de2-846b8cc6ca26:image.png)

`getNextMajorCompactTime`  메서드는 위 설정값에 jitter를 곱한 값으로 지정된다.

![image.png](attachment:d3b6179a-24e9-4cb6-945a-9f31e4f2e251:image.png)

7일 주기인 경우 3.5일 ~ 10.5일 사이에서 랜덤으로 지정된다.

![image.png](attachment:9a3af10a-490d-484d-8ff7-cfcb87a3ab5b:image.png)

`shouldPerformMajorCompaction` 메서드는 메이저 컴팩션 주기 뿐만 아니라 여러 조건들을 확인한다.

컴팩션 대상 파일 수가 하나라면 블록 로컬리티도 임계치에 맞게 조정한다.

---

## 3. 마이너/메이저 컴팩션 결정

`SortedCompactionPolicy` 클래스에서 마이너, 메이저 컴팩션을 결정한다.

![image.png](attachment:fc125735-0a73-4403-b4be-122daceded4e:image.png)

`isUserCompaction`은 유저가 직접 메이저 컴팩션을 요청해야 `true`로 설정된다.

위 코드를 기반으로 살펴보면,

모든 파일을 합치는 메이저 컴팩션의 경우 유저가 요청한 메이저 컴팩션이어야 한다.

시스템이 메이저 컴팩션이 필요하다고 결정하는 경우, 부분적으로 메이저 컴팩션이 일어난다.

![image.png](attachment:bcecfaf2-f35a-4a97-a7d9-ec1ba63b8f11:image.png)

isTryingMajor는 메이저 컴팩션으로 동작할지 여부를 결정하는 플래그다.

조건은, 유저가 요청한 메이저 컴팩션 Or 시스템이 판단한 메이저 컴팩션 필요 여부

시스템에서 메이저 컴팩션 수행이 필요하다고 판단하려면 `shouldPerformMajorCompaction` 및 `candidateSelection.size()` 의 두 가지 조건을 모두 만족해야한다.

`shouldPerformMajorCompaction` 에서는 

1. 마지막 메이저 컴팩션 시간을 확인해서, 그 주기가 돌아왔는지 확인하고.
2. 컴팩션 대상 스토어파일 수가 `comConf.getMaxFilesToCompact()` 보다 작아야 하는데, 이 값은 
`hbase.hstore.compaction.max` 에서 가져온다.

![image.png](attachment:9ed7b7a1-458d-448e-bfc5-6dfc135b66a9:image.png)

즉 시스템에서 메이저 컴팩션이 필요하다고 결정되려면, 메이저 컴팩션 주기가 돌아와야하고, 스토어파일 수가 `hbase.hstore.compaction.max` 보다 작아야 한다. 

![image.png](attachment:c3f6c488-faab-4b7d-b5c8-c2ca69e6289a:image.png)

이후 `CompactionRequest` 객체를 만드는데, 이 때 컬럼패밀리 내부에 참조 스토어파일이 있는 경우 `tryingMajor`가 `true`로 전달되어 메이저 컴팩션으로 동작한다.

![image.png](attachment:4dd5b50d-cc82-4c60-a1ef-6d649973dd04:image.png)

이후 위에서 살펴본 `hbase.hstore.compaction.max` 값만큼 컴팩션 대상 스토어파일 수를 자른다.

하지만 유저가 요청한 메이저 컴팩션의 경우 자르지 않고 모든 파일을 하나로 합친다.

즉, 시스템이 메이저 컴팩션이 필요하여 요청한다고 하더라도 모든 파일을 합치지는 않는다.

(최대 `hbase.hstore.compaction.max` 만큼의 파일을 하나로 합침)

- 문제가 됐던 여러 테이블 중 하나의 스토어파일 목록

![image.png](attachment:6fac9d57-e648-42fb-85e4-b0636bbe4288:image.png)

위 테이블의 경우,

20T 단일 스토어파일과, 여러 참조 스토어파일로 이루어져 있었는데,

이 테이블의 경우 스토어(컬럼 패밀리)에 참조 스토어파일이 있어서 `tryingMajor = true`로 변경었을거고

유저가 요청한 컴팩션이 아니기에 `removeExcessFiles` 메서드에서  `comConf.getMaxFilesToCompact()` 

값 만큼 스토어파일이 subList 되어 부분적으로 메이저 컴팩션이 실행되었을 걸로 추정된다.

테라바이트 단위의 스토어파일이 생기는 이유는,

```java
# 크기가 큰 리전 start key / end key
P_M_2V2_M30S20_A_I_20240319_221644_00000@P300709_00021745_15_srcimg_1997 
P_M_2V2_M30S20_A_I_20241005_025246_00000@P300709_00031156_08_srcimg_3242
```

타임스탬프 기반 RowKey로 인해 항상 테이블의 마지막 리전에 write가 집중된다.

이로 인해 flush가 반복적으로 발생하며 일정 크기의 StoreFile이 다수 생성된다.

StoreFile 수가 증가하면 `hbase.hstore.blockingStoreFiles` 기준을 초과하게 되고,

HBase는 split보다 compaction을 우선해야 하는 상태로 판단하여

system compaction을 재귀적으로 요청하며 split은 지속적으로 지연된다.

즉, 컴팩션이 파일을 “줄이는 속도”보다, flush가 파일을 “만드는 속도”가 더 빠르다.

그래서 compaction → split으로 못 가고 계속 compaction에 묶이는 상태 지속.

결과적으로 크기가 큰 스토어파일 생성.

---

## 4. 컴팩션 대상 파일 셀렉트

`RatioBasedCompactionPolicy` 에서 컴팩션 대상 파일을 선정하는 로직을 확인할 수 있다.

![image.png](attachment:f0f7aa36-956d-4c68-92c4-7835ffe3ea48:image.png)

`tryingMajor` 값은 유저가 요청한 메이저 컴팩션이거나, 참조가 존재하는 컬럼패밀리의 경우 `true`로 전달되는데, 이 경우에는 별도의 파일 선정을 하지 않는다.

`tryingMajor` 가 false인 경우 if 문 내부로 진입해서 파일 크기를 비교하는 로직이 실행된다.

스토어파일을 합침으로써 얻는 이득을 계산한다.

![image.png](attachment:b1f9b6fc-520c-45e4-be21-18338a614090:image.png)

아래 표에서 살펴보면,

| 파일 이름 | 크기 (Size(F)) | 생성 순서 (오래됨 → 최신) |
| --- | --- | --- |
| F1 | 10 MB | 1st (가장 오래됨) |
| F2 | 10 MB | 2nd |
| F3 | 20 MB | 3rd |
| F4 | 50 MB | 4th |
| F5 | 100 MB | 5th |
| F6 | 120 MB | 6th |
| F7 | 10 MB | 7th (가장 최신) |

![image.png](attachment:074cf4f6-887e-40b9-aae9-8f2483c12d97:image.png)

파일을 합침으로써 얻는 이득을 생각하는 로직이다.

여기서 `hbase.hstore.compaction.ratio` 값이 사용된다.

![image.png](attachment:8a773031-b14d-49a3-a51a-1f2ab6b5799f:image.png)

용량이 큰 단일 스토어파일의 경우 나머지 스토어파일들의 합 * ratio 값이 해당 단일 스토어파일보다 커야

컴팩션 대상으로 선정된다. 

→ 큰 스토어파일은 오랫동안 합쳐지지 않을 수 있음.

![image.png](attachment:dbe024c8-e812-4b4c-bf61-2236a36328b4:image.png)

`hbase.hstore.compaction.min.size` 값 이하의 스토어파일들은 위의 ratio 정책을 적용하지 않고, 컴팩션 대상으로 바로 집어넣는다. 해당 설정값보다 큰 스토어파일만 위 로직을 탄다.

---

## 5. 리전 스플릿 조건

### 조건0. 컴팩션이 호출되지 않으면 스플릿도 호출되지 않는다.

![image.png](attachment:21ca09cf-cb1a-4085-8e4c-53bd0bc9bb87:image.png)

`s.needsCompaction() == false`

`s.shouldPerformMajorCompaction() == false` 라면

아무 동작도 일어나지 않는다.

![image.png](attachment:654f7b98-b908-497f-948b-b9722e3b46cb:image.png)

`hbase.hstore.compactionThreshold` (minFilesToCompact) 값이 3 이면,

3 보다 작은 스토어파일 수를 가진 스토어(컬럼 패밀리)는 `s.shouldPerformMajorCompaction()` 로 메이저 컴팩션 여부를 확인한다.

이전 메이저 컴팩션 시간을 보고, 주기가 아직 돌아오지 않아 메이저도 필요 없으면 그냥 `chore()` 종료

→ split도 호출 안 됨

### 조건1. `hbase.hstore.blockingStoreFiles` 값보다 스토어파일 갯수가 적어야한다.

![image.png](attachment:e0c8da33-6f12-4dea-b5b5-db023771c8c8:image.png)

컴팩션 종료 후 스토어(컬럼 패밀리)의 priority를 확인한다.

이 값이 1 이상이여야 스플릿이 호출된다.

0 이하일 경우 스플릿보다 컴팩션이 우선이라고 판단하여 재귀적으로 스플릿을 요청한다.

![image.png](attachment:f0d489ca-6494-481e-9110-c444d4611ddc:image.png)

priority 값은 `blockingFileCount - storefiles.size()` 으로 결정된다.

![image.png](attachment:24c52e50-3a82-442e-b2ad-b9c39eef81b0:image.png)

`hbase.hstore.blockingStoreFiles` 설정값 - 단일 컬럼 패밀리에 존재하는 스토어파일 수 

위 값이 음수인 경우, 스토어파일이 임계치보다 많은 상태로 판단, 

컴팩션 먼저 수행하여 파일을 줄여야 한다고 판단하여 시스템 컴팩션이 재귀적으로 호출된다.

### 조건 2. `hr.getCompactPriority() >= PRIORITY_USER)`

![image.png](attachment:52365972-83a2-4e59-a74f-f60e9903fa5a:image.png)

`requestSplit()` 메서드가 실행됬다는 건, `CompactionPriority`가 양수라는 것.

즉 위의 조건1 에 부합하는 상황이다.

if 문 우측 조건인 `hr.getCompactPriority() >= PRIORITY_USER` 부터 살펴보면,

![image.png](attachment:b21e8f46-0fd0-4b87-bbc1-eff8f4b51b7a:image.png)

`hr.getCompactPriority`는 스토어(컬럼 패밀리)에서 가장 작은 priority를 가져온다.

가져온 priority가 PRIORITY_USER 값인 1 이상이여야 하는데

즉 스토어(컬럼 패밀리) 모두가 조건1 에 부합하여야 한다.

하지만 현재는 스토어 하나만 쓰니까 의미 없다. `shouldSplitRegion()` 만 따지면 된다.

### 조건3. `shouldSplitRegion`() : 리전 갯수 임계치 확인

![image.png](attachment:d6140fa7-2fde-457a-8b9d-2c986acd8af0:image.png)

위 메서드가 true를 반환하려면 리전 개수가 임계치 아래여야 한다.

![image.png](attachment:f58839b0-2a31-4de0-918c-d06b549fa3c9:image.png)

온라인 리전 수가 0.9 * `hbase.regionserver.regionSplitLimit` 값을 초과하는 경우 WARN 로깅. 
`hbase.regionserver.regionSplitLimit` 값을 초과하는 경우 false 반환, 자동 스플릿 X

![image.png](attachment:2aba1e8d-8788-4cd1-b694-acf16b19379b:image.png)

이후에는 스플릿 대상 리전이 충분히 큰지 확인한다.

특이한 점이, 같은 테이블에 속한 리전들의 크기를 보고 점진적으로 스플릿 기준 크기를 증가시킨다는 점.

![image.png](attachment:3f7face8-d341-42bd-b762-b3d2778721cd:image.png)

같은 테이블에 속한 리전들이 많아지게 되면 Math.min에 따라 결국 `hbase.hregion.max.filesize` 로 최대값이 정해진다.

---

## 6. 스플릿 포인트 계산

![image.png](attachment:feb05b72-b8b2-406b-aa14-639f20b9b630:image.png)

가장 큰 스토어파일을 찾아 그 중간 키를 반환한다.

스토어별로 가장 큰 스토어파일을 찾고, 그 스토어파일의 중간 키를 반환한다.

수동으로 스플릿 키 지정 시 -> 계산하지 않고 `getExplicitSplitPoint` 반환.

---

## 7. 스플릿 유저 직접 호출

![image.png](attachment:929dd7d3-1561-4101-8dda-27b4d31b565f:image.png)

- 유저 직접 호출 시 force true 반환

![image.png](attachment:ee3c9c96-e4b4-4d30-9e9d-1d46bb01b73d:image.png)

- 사용자가 직접 미드키를 명시하는 경우 `explicitSplitPoint` 반환.

![image.png](attachment:c9a0f631-822b-46aa-97c5-28eb6a95b443:image.png)

![image.png](attachment:05077181-6ed8-46a8-8534-06cca73009c1:image.png)

사용자가 직접 split point를 명시하는 경우, `forceSplit` 메서드 호출.

내부에서 `explicitSplitPoint` 설정.
