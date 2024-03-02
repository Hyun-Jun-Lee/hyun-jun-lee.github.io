---
title: "TestClient.delete() doesn't support payloads"
date: 2024-03-02
category: [PYTHON]
tag: [python, pytest]
---

프로젝트의 test code를 작성하던 중 아래와 같은 에러가 발생했다.
test code를 작성하던 API endpoint는 아래와 같다.

```python
@router.delete("/ban")
def unban_with_ban_obj(ban_ids: List[int], db: Session = Depends(get_db)):
    """
    차단 해제(설정-차단해제)
    """
    for ban_id in ban_ids:
        check_obj = crud.get_object_or_404(db=db, model=system_model.Ban, obj_id=ban_id)
        obj = crud.ban.remove(db=db, id=ban_id)
    return status.HTTP_204_NO_CONTENT
```

API 개발을 모두 마친 후 테스트 코드를 작성하다 보니, TDD의 중요성을 절실히 깨닫고 있다.
일단, unban_with_ban_obj에 ban_ids 파라미터를 전달하기 위해 `TestClient.delete(url, json=json_data)`와 같이 payloads에 ban 객체의 id를 넣었는데 아래와 같은 에러가 발생했다.

```python
TypeError: TestClient.delete() got an unexpected keyword argument 'json'
```

처음에 든 생각은 '음? delete() method만 키워드 인자가 다른건가?' 였다. 그래서 `delete()` 함수 소스코드를 찾아보았다.

```python
    def delete(  # type: ignore[override]
        self,
        url: httpx._types.URLTypes,
        *,
        params: typing.Optional[httpx._types.QueryParamTypes] = None,
        headers: typing.Optional[httpx._types.HeaderTypes] = None,
        cookies: typing.Optional[httpx._types.CookieTypes] = None,
        auth: typing.Union[
            httpx._types.AuthTypes, httpx._client.UseClientDefault
        ] = httpx._client.USE_CLIENT_DEFAULT,
        follow_redirects: typing.Optional[bool] = None,
        allow_redirects: typing.Optional[bool] = None,
        timeout: typing.Union[
            httpx._client.TimeoutTypes, httpx._client.UseClientDefault
        ] = httpx._client.USE_CLIENT_DEFAULT,
        extensions: typing.Optional[typing.Dict[str, typing.Any]] = None,
    ) -> httpx.Response:
        redirect = self._choose_redirect_arg(follow_redirects, allow_redirects)
        return super().delete(
            url,
            params=params,
            headers=headers,
            cookies=cookies,
            auth=auth,
            follow_redirects=redirect,
            timeout=timeout,
            extensions=extensions,
        )
```

위 코드에서 보면 데이터를 담을만한 인자는 'params' 뿐인데, 인자는 URL 쿼리 파라미터를 지정하는데 사용 된다고 한다.
여기서 나는 아, DELETE Method에서 여러 객체를 한번에 삭제하려고 할때 payloads에 담아서 보내는 것은 RESTFull 하지않은 방법이구나! 라고 생각했고, 그래서 TestClient에서도 delete 메서드에는 payloads를 지원하지 않는구나라고했지만 일단 방법을 찾아보았다.


https://github.com/tiangolo/fastapi/discussions/8409?source=post_page-----8ae12e153a3a--------------------------------


위 글에서 찾은 방법은 `client.METHOD(url)` 대신, `client.requests(METHOD, URL)`을 사용하는 것이었다. FastAPI 공식문서를 찾아보니 `client.requests`를 사용하는 방법에서 `client.METHOD`를 사용하는 방법으로 업그레이드 된 듯 하다. 

이렇게 방법을 찾아서 해결 했지만, 기존 API Endpoint를 RESTFull API 설계원칙에 맞게, 쿼리 파라미터로 정보를 받을 수 있게 변경해보았다.

```python
@router.delete("/ban")
def unban_with_ban_obj(ban_ids: List[int] = Query(..., min_length=1), db: Session = Depends(get_db)):
    """
    차단 해제(설정-차단해제)
    """
```

위 코드와 같이 `Query()`를 작성해서 쿼리파라미터 임을 명시하고 `...`으로 필수 파라미터임을 명시할 수 있다.

갑자기, 내가 사용해봤던 다른 python web framework인 Django에서는 어떻게 햇었지 생각이 나서 예전 프로젝트를 찾아보면서 한번 개발해보았다.

```python
class BanViewSet(ModelViewSet):
    queryset = Ban.objects.all()
    serializer_class = BanSerializer

    def delete(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        ids = serializer.data['ids']
        Ban.objects.filter(pk__in=ids).delete()

        return Response({"message": "차단 해제 완료!"})

    @action(detail=False, methods=['delete'])
    def unban_multiple(self, request):
        ids = request.data['ids']
        Ban.objects.filter(pk__in=ids).delete()

        return Response({"message": "차단 해제 완료!"})
```