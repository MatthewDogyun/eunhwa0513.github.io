---
layout: post
title: 게임 유튜브 페이지에서 게임 타이틀 찾기
subtitle: 게임 유튜브 페이지에서 게임 타이틀 찾기
author: 전지호
categories: 머신러닝
tags: [docker,kubernetes,jupyter,aws]
---
# 게임 유튜브 페이지에서 게임 타이틀 찾기

> 게임 유튜브 영상 페이지에서 게임 정보가 포함되어 있는 경우에 이를 자동으로 수집하는 일을 하고 싶었습니다 🐙

![유튜브 게임정보](https://solution-userstats.s3.amazonaws.com/techblogs/seungjin/2021-06-24/1.png)

유튜브 게임정보

이것을 얻어 올 것입니다.



## 시작하기

----------

크롤링을 처음부터 하려면 수집 소스 코드가 겹쳐서 걱정이었는데, 다행히 pytube에서도 url에 대한 모든 메타데이터에 대한 정보를 저장하고 있었습니다.

```bash
    yt = YouTube(url)
    yt.initial_data['contents']

```

하면 수만가지의 페이지 데이터들이 복잡하게 있습니다.



### 게임 타이틀 데이터도 있나?

----------

이 메타 데이터에서 게임 타이틀에 대한 정보도 있는지 먼저 찾아봤습니다.

![메타데이터](https://solution-userstats.s3.amazonaws.com/techblogs/seungjin/2021-06-24/2.png)

메타데이터

다행히 있네요. 데이터를 읽어오도록 처리하면 될 것 같습니다.

## dict에서 부모 key 역추적하기

----------

> 값에 접근하는 키 일일이 찾기가 너무 귀찮아요…

다행히 value를 기반으로 키값 찾는 코드가 있었습니다.

👉  [링크](https://stackoverflow.com/questions/15210148/get-parents-keys-from-nested-dictionary)

```python
example_dict = { 'key1' : 'value1',
                 'key2' : 'value2',
                 'key3' : { 'key3a': 'value3a' },
                 'key4' : { 'key4a': [{ 
                                         'key4aa': 'value4aa',
                                         'key4ab': 'value4ab',
                                         'key4ac': 'value4ac'
                                     }],
                            'key4b': 'value4b'
                           }
                   } 
                   
def breadcrumb(json_dict_or_list, value):
  if json_dict_or_list == value:
    return [json_dict_or_list]
  elif isinstance(json_dict_or_list, dict):
    for k, v in json_dict_or_list.items():
      p = breadcrumb(v, value)
      if p:
        return [k] + p
  elif isinstance(json_dict_or_list, list):
    lst = json_dict_or_list
    for i in range(len(lst)):
      p = breadcrumb(lst[i], value)
      if p:
        return [i] + p

print(
    breadcrumb(example_dict, 'value4aa')
)
# 결과
# ['key4', 'key4a', 0, 'key4aa', 'value4aa']

```

읽어보면 그냥 무한히 자식 노드로 타고 타고 가다가 값을 발견하면 그간 모아 온 부모 리스트를 리턴한다고 보면 될 것 같습니다. 슥 읽어만 봐서는 value가 중복되면 그중에 하나만 찾고 끝날 것 같은 문제가 있어보이지만, 어쨌든 지금 쓰기에는 알맞는 코드 같습니다.

```css
['twoColumnWatchNextResults', 'results', 'results', 'contents', '1', 'videoSecondaryInfoRenderer', 'metadataRowContainer', 'metadataRowContainerRenderer', 'rows', '0', 'richMetadataRowRenderer', 'contents', '0', 'richMetadataRenderer', 'title', 'simpleText', 'Lost Ark']

```

아까의 yt.initial_data[‘contents’]에서 ‘Lost Ark’를 찾으려면 이렇게 부모들을 타고 가면 나온다고 하네요… 일단 접근 연산자로 이어 붙여봅니다.

```go
parents_list = breadcrumb(yt.initial_data['contents'], 'Lost Ark')

parent_str = ""
for parent in parents_list:
    if type(parent) == int:
        parent_str = parent_str + '[' + str(parent) + ']'
    else:    
        parent_str = parent_str + '[\'' + parent + '\']'

print(parent_str)

## 결과
## ['twoColumnWatchNextResults']['results']['results']['contents'][1]['videoSecondaryInfoRenderer']['metadataRowContainer']['metadataRowContainerRenderer']['rows'][0]['richMetadataRowRenderer']['contents'][0]['richMetadataRenderer']['title']['simpleText']['Lost Ark']

```


결론적으로는 pytube에서 읽어들인 메타데이터에서 다음과 같이 접근하면 게임명이 나오게 됩니다.

```python
def get_game_title(url):
    yt = YouTube(url)
        try:
        game_title = yt.initial_data['contents']['twoColumnWatchNextResults']['results']['results']['contents'][1]['videoSecondaryInfoRenderer']['metadataRowContainer']['metadataRowContainerRenderer']['rows'][0]['richMetadataRowRenderer']['contents'][0]['richMetadataRenderer']['title']['simpleText']
    except KeyError as ex:
        game_title = None
    
    return game_title

print(get_game_title("https://www.youtube.com/watch?v=S3ek945d1xE"))

# 결과
# 'Lost Ark'

```

중간의 0이나 1같은 상수 접근이 좀 부담스럽긴 한데, 일단 영상 속에서 게임에 대한 태그가 1개만 등록되어 있다면 무리없이 사용되는 것 같습니다. 정교한 크롤링을 위해서는 구조를 더 분석해야겠지만 일단 이 형태로 운용해보려고 합니다.

## 맺음말

----------

영화, 소설 등 게임이 아닌 경우 케이스들을 검색해봤는데, 이런 추가 데이터들이 있지는 않았습니다. 유독 게임과 음악(얘는 저작권 때문에..)에만 잘 카테고리화 되어있는 것 같습니다. 잘은 모르지만, 아마 게임의 경우 업로더가 직접 설정하는게 아닌가 싶은데, 아무튼 없을 수도 있는 데이터라 주의해야 합니다. 하지만, 있는 경우에 한해서 영상 분류에 큰 도움이 되므로 사용하면 좋을 것 같습니다.
