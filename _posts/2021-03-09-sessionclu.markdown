---
title: MSA 환경에서 발생하는 세션 클러스터링 문제에 대한 고민과 해결방안
subtitle: Spring,Web,MSA
date: 2021-01-29
categories:
  - Development
tags:
  - Spring
---

## 세션은 왜 사용하는것일까
http 통신은 기본적으로 요청에 대한 응답으로 통신이 끝난다, 즉 Stateless 한 통신이다.
하지만 비즈니스로직을 실제로 구현하다보면 Statefull해야만 구현 가능한 요소들이 존재한다. 예를들면 장바구니 서비스에서 로그인하지 않은 사용자의 임시 장바구니에 저장된 목록들이 있을때, 서버가 Stateless 하다면 장바구니에 단 하나의 상품만 담기고 페이지 변환을 했을때 다시 그 목록을 불러올 수 없을것이다. 즉, 클라이언트든 서버든 이런 사용자가 가져야 할 일련의 상태정보를 가지고 있어야한다는 것이다.

즉 위의 예시에서 장바구니가 계속 유지되려면 목록에 대한 상태가 지속적으로 관리되어야한다 따라서 그런 stateful한 구조가 필요했기 때문에 cookie와 Session의 개념이 도입되었다.

## Spring에서의 Session

Session은 클라이언트가 서버에 어떤 요청을 보냈을때, 클라이언트의 cookie내에 들어있는 jsessionid 라는 attribute의 value를 기준으로 Servlet 단에서 자동으로 생성되게 된다, 자주 사용하는 WAS인 Tomcat의 conf/web.xml에 default timeout이 30분으로 설정되어있다. 

Spring에서 세션을 핸들링하는 방법은 크게 두가지가 있다.
-  

##### HttpServletRequest에서 HttpSession을 핸들링하는 방법
- getSession() 메서드를 이용해 현재 요청한 클라이언트의 세션 인스턴스인 HttpSession을 핸들링 할 수 있다. 그리고 HttpSession.inValidate() 명령어를 통해 세션을 서버에서 삭제할 수 있다.

##### HttpSession을 파라미터로 바로 받는법
- RequestMapping 어노테이션을 적용한 컨트롤러의 메소드의 파라미터로 HttpSession 타입을 추가하면 인스턴스가 맵핑되어 핸들링할 수 있다.


###### RequestMapping 어노테이션에서 파라미터를 넣었을때 어떻게 매핑이 될까?
우리가 컨트롤러 내에서 @RequestMapping 어노테이션을 선언하면 해당 메소드는 RequestMappingHandlerAdapter의 구현체에 있는 HandlerMethodArgumentResolver의 구현체인 HandlerMethodArgumentResolverComposite 클래스 인스턴스를 통해 해당 메소드에 맞는 적절한 parameter를 매핑해준다.

아래는 Spring의 default HandlerMethodArgumentResolver 메소드의 구현이다. Spring은 디폴트로 사용하는 리졸버들의 리스트이다

```java
/**
	 * Return the list of argument resolvers to use including built-in resolvers
	 * and custom resolvers provided via {@link #setCustomArgumentResolvers}.
	 */
	private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();

		// Annotation-based argument resolution
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		resolvers.add(new ServletModelAttributeMethodProcessor(false));
		resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new RequestHeaderMapMethodArgumentResolver());
		resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new SessionAttributeMethodArgumentResolver());
		resolvers.add(new RequestAttributeMethodArgumentResolver());

		// Type-based argument resolution
		resolvers.add(new ServletRequestMethodArgumentResolver());
		resolvers.add(new ServletResponseMethodArgumentResolver());
		resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RedirectAttributesMethodArgumentResolver());
		resolvers.add(new ModelMethodProcessor());
		resolvers.add(new MapMethodProcessor());
		resolvers.add(new ErrorsMethodArgumentResolver());
		resolvers.add(new SessionStatusMethodArgumentResolver());
		resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

		// Custom arguments
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
		resolvers.add(new ServletModelAttributeMethodProcessor(true));

		return resolvers;
	}
```

실제로 HandlerMethodArgumentResolver의 구현체는 HandlerMethodArgumentResolverComposite 이고 여기서 실제 맵핑을 수행하게 된다. 코드는 아래와 같다.


```java
public class HandlerMethodArgumentResolverComposite implements HandlerMethodArgumentResolver {

    ...
    private final List<HandlerMethodArgumentResolver> argumentResolvers = new LinkedList<>();

    private final Map<MethodParameter, HandlerMethodArgumentResolver> argumentResolverCache =
            new ConcurrentHashMap<>(256);

    @Override
    @Nullable
    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

        HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
        if (resolver == null) {
            throw new IllegalArgumentException(
                    "Unsupported parameter type [" + parameter.getParameterType().getName() + "]." +
                            " supportsParameter should be called first.");
        }
        return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
    }

    @Nullable
    private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
        HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
        if (result == null) {
            for (HandlerMethodArgumentResolver methodArgumentResolver : this.argumentResolvers) {
                if (methodArgumentResolver.supportsParameter(parameter)) {
                    result = methodArgumentResolver;
                    this.argumentResolverCache.put(parameter, result);
                    break;
                }
            }
        }
        return result;
    }
}
```


## 100만명의 사용자가 서비스에 요청을 보내고 있다면?
세션은 기본적으로 서버의 in-memory 방식으로 저장한다. 인스턴스에 존재하고 클라이언트의 쿠키에 담겨있는 jsessionid 를 통해 request가 무슨 세션을 갖고있는지 알 수 있다. 근데 100만개의 인스턴스가 JVM의 힙 영역에 존재하려고 까불어 재킨다면.. 스택오버플로우는 따놓은 당상이다.

어떻게 이 문제를 해결할 수 있을까?

### 파일으로 jsessionId에 대한 세션정보를 저장하기. 
파일으로 각 사용자에 대한 세션 정보를 저장할 수 있다. 각 세션마다 스토리지에 jsessionid에 해당하는 세션정보를 파일로서 저장하면 된다. 

- ##### 세션을 파일으로 저장하는 방식에서 발생하는 오류
    파일으로 세션을 저장하면 발생하는 오류는 서비스 인스턴스가 두개가 되었을때 발생한다. 즉 별도의 WAS에서 같은 서비스가 2개이상으로 가용되고, 요청이 서버상황에 따라 별도의 WAS에서 돌아간다면 즉, A와 B라는 WAS에 서비스가 배포되었을떄, A에 요청을 했을때 A WAS에 Session 파일이 저장되는데, 다음 요청이 클라이언트 로드밸런서 또는 L4 스위치의 로드밸런서에 의해 B로 가게 된다면? B WAS에는 해당 SessionFile이 존재하지 않고, 새로운 세션이라 판단하게 된다. 즉 각 서버마다 세션 동기화 문제가 발생하게 된다.


### Tomcat의 세션 클러스터링 기능을 이용하기
위와같이 세션 동기화 문제 또는 세션의 관리에 대한 문제를 해결하기위해 세션 클러스터링을 수행해야한다.

[Tomcat의 자체내장 세션 클러스터링 기능](https://oingdaddy.tistory.com/149)을 자체적으로 제공하고 있다. 자세한 사용방법은 호형 님의 tistory 글으로 대체한다.

- ##### Tomcat의 Session clusturing 기능을 이용하는 방법의 문제점 
    이 방법에서는 클러스터링 configration을 WAS별로 설정을 지속적으로 해줘야 한다. 그러나 현재 웹서비스의 배포형태가 MSA 형태로 구성되어있다면 서비스 인스턴스가 지속적으로 변경되어질텐데 인스턴스가 늘어날때마다 지속적으로 WAS 설정을 해야한다는 단점이 있다.

### ☀️ 세션을 저장하는 별도의 Database를 사용하는 방법 (Best practice)


