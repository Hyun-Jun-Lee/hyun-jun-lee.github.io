---
title: "짠내나는 데이터 다루기" #Article title.
date: 2023-12-17
category: [PYTHON, PYCORN] #One, more categories or no at all.
tag: [pycorn, python]
---

__PyCon korea 2023 박조은__

if 메모리가 8G인 환경에서 32G 이상의 큰 파일을 메모레에 모두 로드해서 사용해야한다면?

예산이 충분하다면 메모리를 늘리는게 가장쉬운 방법이겠지만, 그렇지 않은 상황에서 최대한 메모리 사용량을 줄여 작업활 수 있는 환경을 만들어야한다.

# 메모리 용량 확보

- 현재 실행 중인 프로그램 중 필요하지 않는 것들 모두 삭제 하여 메모리 확보
- 백그라운드 프로그램 제거 or 비활성화
- 브라우저 캐시 기록 삭제 및 제거
- 가장 좋은 방법은 컴퓨터 재부팅!

# 메모리 사용량을 줄이기 제안

- 샘플링(행 줄이기)
- 특정 열 인덱싱과 선택적 로딩(열 줄이기)
- 청크, 반복문
- 데이터 타입 변경
- parquet 형식으로 압축
- 병렬처리
- DB활용(NoSQL)
- 분산 처리 프레임 워크 dask, pyspark 등 활용

# 큰 데이터셋을 나눠서 가져오기

## 데이터 샘플링

- 무작위로 선택된 데이터 샘플링(`df.sample(n)`, `df.sample(frac=01)`
- 도메인 정보에 따라 샘플링 기준 정하기
    - ex) 특정 기간, 특정 고객군 등

## 필요한 데이터만 서브셋으로 가공

- 우리에게 필요한 데이터는 모든 데이터가 아님
- 필요없는 컬럼, 로우는 제거하고 필요한 데이터에서만 샘플링

## Chunked Processing

- 데이터를 메모리에 한번에 모두 로드하는게 아니라, 청크 단위로 반복적으로 처리
- pandas의 `read_csv()`같은 메서드에서 chunksize 파라미터를 제공함
- Iterator과 함께 사용해서 chink_size만 가져와서 작업 후 저장하고, 다음 chunk_size 처리

## Parquet

- 효율적인 데이터 저장 및 검색을 위해 설계된 오픈소스
- 보통 csv, text 파일들을 행 기반으로 저장하는데, parquet에서는 열을 기준으로 저장하고 보통 같은 열에 있는 데이터들은 모두 같은 데이터타입이기 때문에 압축 성능이 매우 좋다.
- 특정 열 값의 데이터를 가져오는 쿼리는 전체 행 데이터를 읽어올 필요가 없어서 성능이 좋다.
- 열 기준으로 작업을 처리하기 때문에, 각 열의 타입에 따른 인코딩 기술적용 가능
- 큰 데이터셋을 파일로 분할하여 병렬처리도 가능
- metadata에서 데이터의 유형 및 통계 관련 정보를 제공

## Pandas 데이터 타입별 크기

![image](https://github.com/BIS-KIT/BISKIT-Backend/assets/76996686/a82bfe7e-5cea-4524-939c-7f9066630f9a)

파란색으로 색칠된 타입들이 pandas에서 read_csv와 같은걸로 데이터셋을 읽어 올 때 default로 지정된 데이터 타입인데, 이것들을 줄여주기만 해도 메모리를 매우 절약할 수 있음


## csv에서 파일을 읽어올 때

매우 큰 데이터셋의 경우 chunksize를 지정해서 초반의 어느 정도만 가져와서 어떤 데이터들이 있는지 확인해보고, 아래 예시와 같이 미리 데이터 타입을 지정해 주는 것이 효율적

![image](https://github.com/BIS-KIT/BISKIT-Backend/assets/76996686/4433d7cf-d000-4d44-93e1-89a57ae701dc)

# 대용량 데이터 영량 줄이기 예시

공공데이터포털에서 `국민건강보험공단_의약품처방정보` 데이터를 09년부터 20년까지 모두 다운받으면 31GB 정도 된다.

```
def downcast(df_chunk):
    for col in df_chunk.columns:
        dtypes_name = df_chunk[col].dtypes.name
        if dtypes__name.startswith("float"):
            df_chunk[col] = pd.to_numeric(df_chunk[col], downcast="float)
        elif dtypes_name.startswith("int"):
            # 최소값 구해서 음수가 있을 경우 integer
            # 음수가 없을 경우 unsigned
            if df_chunk[col].min() < 0:
                df_chunk[col] = pd.to_numeric(df_chunk[col], downcast="integer")
            else:
                # unsigend 타입은 0과 양수의 정수만 저장 가능한 타입
                df_chunk[col] = pd.to_numeric(df_chunk[col], downcast="unsigend")
        elif dtypes_name.startswith("object"):
                # 문자일 때는 category로 변경
                # 한정된 변수의 반복되는 값들을 효율적으로 저장하기위한 pandas의 특수한 데이터 타입
                df_chunk[col] = df_chunk[col].astype("category")
    return df_chunk

def downcast_csv_to_parquet(zip_file_name):
    start = time.time()
    # 정규표현식으로 파일 이름 추춢
    result = re.findall(r'data/(.*?\.csv', zip_file_name, re.IGNORECASE))
    save_file_name = result[0]

    chunk_size = le6
    chunk_iter = pd.read_csv(zip_file_name, chunksize=chunk_size)
    row_count = 0
    chunk_list = []

    for chunk in chunk_iter:
        row_count = row_counut + chunk.shape[0]
        df_chunk = downcast(chunk)
        # downcast 후에 list에 append 해두었다가 concat 해도 되지만
        # 메모리 부족을 대비해 파일로 저장 후 불러오는 방법을 사용
        df_chunk.to_parquet(f'data_parquet/{save_file_name}-{df_chunk.index[0]}-{df_chunk.index[-1]}.parquet', index=False)
    end = time.time()
    return f'{save_file_name} {row_count} {end-start: .0f}

```

- `pandas.to_numeric` : float64->float32, int64->int32 와 같이 데이터타입을 더 작은 타입으로 변경
    ![image](https://github.com/BIS-KIT/BISKIT-Backend/assets/76996686/7c126e12-3809-4b52-86cb-ed5d17370bd3)



1. 먼저 chunk_size를 지정해서 데이터 셋을 읽어온다. 
2. chunk_size로 가져온 데이터들을 확인 후 필요없는 열or행 제거 
3. downcast 한 후 메모리에 보관하는 것이 아니라 parquet 파일로 저장
4. 작업이 모두 끝난 후 각각의 파일을 하나의 파일로 합치기

이렇게 큰 용량의 데이터를 chunk_size로 나눈 후 downcast - parquet형식으로 저장하는 것만으로도 용량을 크게 줄일 수 있고, 모든 데이터가 아니라 필요한 특정 열, 행 만 가져온다면 용량을 더 줄일 수 있다.