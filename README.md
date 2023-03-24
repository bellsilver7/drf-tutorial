# Djang REST Framework 튜토리얼

## 목차

- [튜토리얼 1: Serialization](#튜토리얼-1:-Serialization)
- [튜토리얼 2: 요청과 응답](#튜토리얼-2:-요청과-응답)
- [튜토리얼 3: 클래스 기반 뷰](#튜토리얼-3:-클래스-기반-뷰)
    - [클래스 기반 뷰를 사용해 API 다시 작성하기](#클래스-기반-뷰를-사용해-API-다시-작성하기)
    - [mixin 사용하기](#mixin-사용하기)
    - [generic 클래스 기반 뷰 사용하기](#generic-클래스-기반-뷰-사용하기)
- [튜토리얼 4: 인증 및 권한](#튜토리얼-4:-인증-및-권한)
- [튜토리얼 5: 관계 및 하이퍼링크 API](#튜토리얼-5:-관계-및-하이퍼링크-API)
- [튜토리얼 6: 뷰셋 및 라우터](#튜토리얼-6:-뷰셋-및-라우터)

## 튜토리얼 1: Serialization

## 튜토리얼 2: 요청과 응답

## 튜토리얼 3: 클래스 기반 뷰

- 클래스 기반 뷰를 사용해 API 뷰를 작성할 수 있다.
- 일반적인 기능을 재사용하고 코드를 DRY(Don't Repeat Yourself)하게 유지하는 데 도움이 되는 강력한 패턴.

### 클래스 기반 뷰를 사용해 API 다시 작성하기

루트 뷰를 클래스 기반 뷰로 다시 작성한다. 이것은 views.py 의 약간의 리팩터링이다.

```python
# snippets/views.py
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class SnippetList(APIView):
    """
    List all snippets, or create a new snippet.
    """

    def get(self, request, format=None):
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

이전 사례와 매우 유사하지만, 서로 다른 HTTP 방식 간에 더 나은 분리가 가능하다. 또한 views.py 의 인스턴스 뷰를 업데이트해야 한다.

```python
class SnippetDetail(APIView):
    """
    Retrieve, update or delete a snippet instance.
    """

    def get_object(self, pk):
        try:
            return Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        snippet = self.get_object(pk)
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        snippet = self.get_object(pk)
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

지금은 여전히 함수 기반 뷰와 상당히 유사하다.

```python
from django.urls import path
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    path('snippets/', views.SnippetList.as_view()),
    path('snippets/<int:pk>/', views.SnippetDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

개발 서버를 실행하면 이전과 동일하게 작동해야 한다.

### mixin 사용하기

클래스 기반 뷰를 사용하는 가장 큰 이유 중 하나는 재사용 가능한 행동을 쉽게 구성할 수 있다는 것이다.

우리가 지금까지 사용해 온 create/retrieve/update/delete 작업은 우리가 만든 모든 모델 지원 API 뷰와 매우 유사할 것이다.
이러한 일반적인 동작은 REST 프레임워크의 mixin 클래스에서 구현된다.

mixin 클래스를 사용하여 뷰를 구성하는 방법. 여기 우리의 views.py 모듈이 있다.

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import mixins
from rest_framework import generics


class SnippetList(mixins.ListModelMixin,
                  mixins.CreateModelMixin,
                  generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

여기서 잠깐 동작을 살펴보면, GenericAPIView를 사용해 뷰를 구현하고 ListModelMixin과 CreateModelMixin을 추가한다.

기본 클래스는 핵심 기능을 제공하며, mixin 클래스는 .list() 및 .create() 작업을 제공한다. 그런 다음 get 및 post 메서드를 적절한 작업에 명시적으로 바인딩한다.

```python
class SnippetDetail(mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    generics.GenericAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```

SnippetList와 비슷하게 핵심 기능을 제공하기 위해 GenericAPIview 클래스를 사용하고 .retrieve(), .update() 및 .destroy() 작업을 제공하기 위해 mixin을 추가한다.

### generic 클래스 기반 뷰 사용하기

mixin 클래스를 사용해 이전보다 코드를 약간 적게 사용하도록 뷰를 다시 작성했지만 한 단계 더 나아갈 수 있다.
REST 프레임워크는 이미 혼합된 일반 뷰 세트를 제공하여 views.py 모듈을 더욱
축소하는 데 사용할 수 있다.

```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework import generics


class SnippetList(generics.ListCreateAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer


class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
```

코드는 좋고, 깨끗하고, 관용적인 장고처럼 보인다.

## 튜토리얼 4: 인증 및 권한

## 튜토리얼 5: 관계 및 하이퍼링크 API

## 튜토리얼 6: 뷰셋 및 라우터