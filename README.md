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

현재 우리의 API는 누가 코드 스니펫을 편집하거나 삭제할 수 있는지에 대한 제한이 없습니다. 우리는 다음을 확인하기 위해 좀 더 발전된 행동을 하길 원한다:

- 코드 스니펫은 항상 작성자와 관련이 있다.
- 인증된 사용자만 스니펫을 만들 수 있다.
- 스니펫 작성자만 업데이트하거나 삭제할 수 있다.
- 인증되지 않은 요청은 완전한 읽기 전용 액세스 권한을 가져야 한다.

### Snippet 모델에 정보 추가하기

스니펫 모델 클래스를 몇 가지 변경할 것이다. 먼저, 몇 개의 필드를 추가한다. 그 필드 중 하나는 코드 스니펫을 만든 사용자를 나타내는 데 사용될 것이다. 다른 필드는 코드의 강조 표시된 HTML 표현을 저장하는
데 사용될 것이다.

Models.py의 스니펫 모델에 다음과 같이 두 필드를 추가하면 된다.

```python
owner = models.ForeignKey('auth.User', related_name='snippets', on_delete=models.CASCADE)
highlighted = models.TextField()
```

또한 모델이 저장될 때 **pygments** 코드 강조 표시 라이브러리를 사용하여 강조 표시된 필드를 채우는지 확인해야 한다.

우리는 몇 가지 추가 import가 필요할 것이다:

```python
from pygments.lexers import get_lexer_by_name
from pygments.formatters.html import HtmlFormatter
from pygments import highlight
```

그리고 이제 모델 클래스에 .save() 메소드를 추가할 수 있다:

```python
def save(self, *args, **kwargs):
    """
    Use the `pygments` library to create a highlighted HTML
    representation of the code snippet.
    """
    lexer = get_lexer_by_name(self.language)
    linenos = 'table' if self.linenos else False
    options = {'title': self.title} if self.title else {}
    formatter = HtmlFormatter(style=self.style, linenos=linenos,
                              full=True, **options)
    self.highlighted = highlight(self.code, lexer, formatter)
    super().save(*args, **kwargs)
```

다 끝나면 데이터베이스 테이블을 업데이트해야 한다. 일반적으로 데이터베이스 마이그레이션을 만들 수 있지만, 이 튜토리얼의 목적을 위해 데이터베이스를 삭제하고 다시 시작한다.

```shell
rm -f db.sqlite3
rm -r snippets/migrations
python manage.py makemigrations snippets
python manage.py migrate
```

API를 테스트하는 데 사용할 사용자를 생성하기 위해 createsuperuser 명령을 사용해 볼 수 있다.

```shell
python manage.py createsuperuser
```

### 사용자 모델에 대한 엔드포인트 추가

이제 함께 작업할 사용자가 생겼으니, 그 사용자의 표현을 API에 추가하는 것이 좋습니다.
새로운 시리얼라이저를 만드는 것은 쉽다.
Serializers.py에 추가한다:

```python
from django.contrib.auth.models import User


class UserSerializer(serializers.ModelSerializer):
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

    class Meta:
        model = User
        fields = ['id', 'username', 'snippets']
```

'Snippets'는 사용자 모델의 역 관계이기 때문에 ModelSerializer 클래스를 사용할 때 기본적으로 포함되지 않으므로 명시적인 필드를 추가해야 한다.

우리는 또한 views.py에 몇 가지 뷰를 추가할 것이다.
우리는 사용자 표현에 읽기 전용 뷰를 사용하고 싶기 때문에,
ListAPIView와 RetrieveAPIView 일반 클래스 기반 뷰를 사용할 것이다.

```python
from django.contrib.auth.models import User


class UserList(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer


class UserDetail(generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

UserSerializer 클래스도 import 해준다.

```python
from snippets.serializers import UserSerializer
```

마지막으로 URL conf에서 참조하여 아래와 같이 뷰를 API에 추가해야 한다. Snippets/urls.py의 패턴에 추가한다.

```python
path('users/', views.UserList.as_view()),
path('users/<int:pk>/', views.UserDetail.as_view()),
```

### 스니펫을 사용자와 연결하기

지금 당장은 스니펫을 만든 사용자를 스니펫 인스턴스와 연결할 방법이 없어 스니펫을 만들 수 없다. 사용자는 직렬화된 표현의 일부로 전송되지 않고, 대신 들어오는 요청의 속성이다.

그것을 처리하는 방법은 스니펫 뷰에서 .perform_create() 메서드를 재정의하여 인스턴스 저장 관리 방법을 수정하고 들어오는 요청이나 요청된 URL에 암시적인 정보를 처리할 수 있다.

SnippetList 뷰 클래스에서, 다음 함수를 추가한다:

```python
def perform_create(self, serializer):
    serializer.save(owner=self.request.user)
```

serializer의 create() 메서드는 이제 요청의 검증된 데이터와 함께 추가 '소유자' 필드로 전달된다.

### serializer 업데이트하기

이제 스니펫을 만든 사용자와 연결되어 있으므로, 그것을 반영하기 위해 SnippetSerializer를 업데이트한다. Serializers.py의 serializer 정의에 다음 필드를 추가한다:

```python
owner = serializers.ReadOnlyField(source='owner.username')
```

참고: 내부 메타 클래스의 필드 목록에 'owner'도 추가해야 한다.

이 필드는 꽤 흥미로운 일을 하고 있다. source 인수는 필드를 채우는 데 사용되는 속성을 제어하며, 직렬화된 인스턴스의 모든 속성을 가리킬 수 있다. 또한 위에 표시된 point 표기를 사용할 수 있으며, 이
경우 장고의 템플릿 언어와 마찬가지로 주어진 속성을 통과할 것이다.

우리가 추가한 필드는 CharField, BooleanField 등과 같은 다른 입력 필드와 달리 입력되지 않은 ReadOnlyField 클래스다. 입력되지 않은 ReadOnlyField는 항상 읽기 전용이며,
직렬화된 표현에 사용되지만, 역직렬화될 때 모델 인스턴스를 업데이트하는 데 사용되지 않는다. 또한 여기서 CharField(read_only=True)를 사용할 수 있다.

### 뷰에 필요한 권한 추가하기

이제 코드 스니펫이 사용자와 연결되어 있기 때문에, 인증된 사용자만 코드 스니펫을 생성, 업데이트 및 삭제할 수 있는지 확인하길 원한다.

REST 프레임워크에는 주어진 뷰에 접근할 수 있는 사람을 제한하는 데 사용할 수 있는 여러 권한 클래스가 포함되어 있다. 이 경우 IsAuthenticatedOrReadOnly로 인증된 요청이 읽기-쓰기 액세스를
얻고 인증되지 않은 요청이 읽기 전용 액세스를 얻을 수 있도록 한다.

먼저 뷰 모듈에 다음과 같이 import를 추가한다.

```python
from rest_framework import permissions
```

그런 다음, SnippetList와 SnippetDetail 뷰 클래스 모두에 다음 속성을 추가한다.

```python
permission_classes = [permissions.IsAuthenticatedOrReadOnly]
```

### Browsable API에 로그인 추가하기

브라우저를 열고 Browsable API로 이동하면, 더 이상 새로운 코드 스니펫을 만들 수 없다는 것을 알게 될 것다. 그렇게 하기 위해서는 사용자로 로그인할 수 있어야 한다.

프로젝트 수준의 urls.py 파일에서 URLconf를 편집하여 Browsable API와 함께 사용할 로그인 보기를 추가할 수 있다.

파일 상단에 다음과 같이 import 한다:

```python
from django.urls import path, include
```

그리고, 파일의 끝에, Browsable API에 대한 로그인 및 로그아웃 뷰를 포함하는 패턴을 추가한다.

```python
urlpatterns += [
    path('api-auth/', include('rest_framework.urls')),
]
```

패턴의 'api-auth/' 부분은 실제로 사용하고 싶은 URL일 수 있다.

이제 브라우저를 다시 열고 페이지를 새로 고치면 페이지 오른쪽 상단에 '로그인' 링크가 표시된다. 이전에 만든 사용자 중 한 명으로 로그인하면, 코드 스니펫을 다시 만들 수 있다.

몇 개의 코드 스니펫을 만들면, '/users/' 엔드포인트로 이동하고, 표현에는 각 사용자의 '스니펫' 필드에 각 사용자와 관련된 스니펫 ID 목록이 포함되어 있다.

### 개체 수준 권한

우리는 모든 코드 스니펫을 누구나 볼 수 있기를 원하지만, 코드 스니펫을 만든 사용자만 업데이트하거나 삭제할 수 있도록 한다.

그렇게 하려면 사용자 지정 권한을 만들어야 한다다.

스니펫 앱에서 새 파일 permissions.py를 만든다.

```python
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission to only allow owners of an object to edit it.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request,
        # so we'll always allow GET, HEAD or OPTIONS requests.
        if request.method in permissions.SAFE_METHODS:
            return True

        # Write permissions are only allowed to the owner of the snippet.
        return obj.owner == request.user
```

이제 SnippetDetail view 클래스에서 permission_classes 속성을 편집하여 스니펫 인스턴스 엔드포인트에 사용자 지정 권한을 추가할 수 있다.

```python
permission_classes = [permissions.IsAuthenticatedOrReadOnly,
                      IsOwnerOrReadOnly]
```

물론 IsOwnerOrReadOnly 클래스를 import 해야 한다.

```python
from snippets.permissions import IsOwnerOrReadOnly
```

이제 브라우저를 다시 열면 코드 스니펫을 만든 동일한 사용자로 로그인한 경우 'DELETE' 및 'PUT' 작업이 스니펫 인스턴스 엔드포인트에만 나타난다.

### API로 인증하기

이제 API에 대한 권한 세트가 있기 때문에, 스니펫을 편집하려면 요청을 인증해야 한다. 인증 클래스를 설정하지 않았기 때문에, 현재 SessionAuthentication과 BasicAuthentication의
기본값이 적용된다.

웹 브라우저를 통해 API와 상호 작용하면 로그인할 수 있으며, 브라우저 세션은 요청에 필요한 인증을 제공한다.

프로그래밍 방식으로 API와 상호 작용하는 경우 각 요청에 대한 인증 자격 증명을 명시적으로 제공해야 한다.

인증하지 않고 스니펫을 만들려고 하면 오류가 발생한다:

```shell
http POST http://127.0.0.1:8000/snippets/ code="print(123)"

{
    "detail": "Authentication credentials were not provided."
}
```

이전에 만든 사용자 중 한 명의 사용자 이름과 비밀번호를 포함하여 성공적인 요청을 할 수 있다.

```shell
http -a admin:password123 POST http://127.0.0.1:8000/snippets/ code="print(789)"

{
    "id": 1,
    "owner": "admin",
    "title": "foo",
    "code": "print(789)",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}
```

### 요약

이제 웹 API에 대한 상당히 세분화된 권한 세트를 가지고 있으며, 시스템 사용자와 그들이 만든 코드 스니펫에 대한 엔드포인트를 가지고 있다.

튜토리얼의 5부에서 우리는 강조된 스니펫을 위한 HTML 엔드포인트를 만들어 모든 것을 함께 묶고, 시스템 내의 관계에 대한 하이퍼링크를 사용하여 API의 응집력을 개선하는 방법을 살펴볼 것이다.

## 튜토리얼 5: 관계 및 하이퍼링크 API

## 튜토리얼 6: 뷰셋 및 라우터