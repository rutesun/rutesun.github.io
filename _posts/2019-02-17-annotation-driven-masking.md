---
title: 어노테이션 기반으로 개인 정보 마스킹처리하기
description: 어노테이션 기반으로 개인 정보 마스킹처리하기
image: 
category: development
tags: kotlin devday
---
# 개요
개인정보 보안은 개발을 하다보면 신경쓰기 쉽지 않지만 처음부터 잘해야지 나중에 가서 처리하게 되면 부채를 감당하기 힘들다.

db에 저장되는 개인정보는 hash 값을 저장하거나 암호화된 정보를 넣어서 잘 처리되는 경우가 많지만 로그까지 신경 쓰기는 쉽지 않다. 

개인정보는 보통 로그에 저장할 때 마스킹 처리하게 될 텐데 어떻게 하면 로그에 저장할 개인 정보를 마스킹하는 기능을 유지 보수하기 쉽고 스마트하게 처리할 수 있을까?

1. Anno요ation에 개인정보에 해당하는 필드를 마스킹 할지 방법을 주입 받는다.
2. Json으로 serialize할 때 주입 받은 마스킹 함수로 잘 처리해보자.

이렇게 하면 충분히 쉽고 관리하기 편하게 개인정보를 잘 처리할 수 있지 않을까?

아래 모든 예제는 코틀린 기반으로 작성되었다.

# 마스킹 annotaion을 만들어보자.

```kotlin
    interface Masking : (String) -> String {
        override fun invoke(s: String): String = s
    }
   자
    @Target(AnnotationTarget.FIELD, AnnotationTarget.FUNCTION)
    @Retention(AnnotationRetention.RUNTIME)
    annotation class MaskRequired(
        val masker: KClass<out Masking>
    )
```    

일단 개인정보를 입맛대로 마스킹 할 마스킹 함수와 annotation을 만들어 보자. 

어떤 정보를 마스킹 처리할 지 나타낼  `MaskRequired` annotaion과 어떻게 마스킹을 처리할지 정보를 담은 `Masking` 인터페이스를 만들었다.

# Json serialize 할 때 annotation 을 적용하기

이제 json serialize를 할 때 `MaskRequired` 가 있을 때 자동으로 처리하기 위한 커스텀 Serializer를 만들어보자. 

Jackson 라이브러리는 객체를 입맛대로 serialize하기 위해 `Serializer` 라는 인터페이스를 제공하고 그 인터페이스를 구현한 serializer를 jackson에게 알려주면 우리가 원하는대로 json의 결과값을 핸들링할 수 있다.

Serializer에서 어떻게 프로퍼티에 대한 메타 정보를 읽을 수 있을까? 

`ContextualSerializer` 는 아래 설명에서 볼 수 있듯이 어떤 serializer를 사용해야할 지에 대한 선택하기 위해 사용되는 인터페이스고 serializer를 선택하기 위한 메타 정보를 우리에게 알려준다. 
```java
    /**
     * Add-on interface that {@link JsonSerializer}s can implement to get a callback
     * that can be used to create contextual instances of serializer to use for
     * handling properties of supported type. This can be useful
     * for serializers that can be configured by annotations, or should otherwise
     * have differing behavior depending on what kind of property is being serialized.
     */
```
`ContextualSerializer`  와 `StdSerializer` 를 상속받아 우리가 계획했던 모든 기능이 완성되었다. 
```kotlin
    class MaskingSerializer(val masker: (String) -> String = { s -> s }) : StdSerializer<String>(String::class.java), ContextualSerializer {
    
    
        override fun serialize(value: String, gen: JsonGenerator, provider: SerializerProvider) {
            gen.writeString(masker(value))
        }
    
    
        override fun createContextual(provider: SerializerProvider, property: BeanProperty): JsonSerializer<*> {
            val ann = property.getAnnotation(MaskRequired::class.java)
            if (ann != null) {
                var masker = ann.masker.objectInstance ?: ann.masker.createInstance()
                return MaskingSerializer(masker)
            }
            return provider.findKeySerializer(property.type, property)
        }
    }
```
잘 동작하는지 아래와 같이 샘플 클래스를 만들어서 테스트를 해보자

```kotlin
    class UserTest {
        object NameMasker : Masking {
            override operator fun invoke(s: String): String = Regex("(?<=.{1}).*").replace(s, "*")
        }
    
        data class User(
            @JsonSerialize(using = MaskingSerializer::class)
            @MaskRequired(masker = NameMasker::class)
            val name: String = ""
        )
    
        @Test
        fun `simple masking`() {
            val user = User("홍길동")
            val jsonStr = objectMapper.writeValueAsString(user)
            assertEquals("""{"name":"홍**"}""".trimIndent(), jsonStr)
        }
    }
```
테스트를 위해 이름을 마스킹 처리 하기 위해 `Masking` 인터페이스를 상속한 NameMasker라는 클래스를 만들었다. 이 클래스는 이름의 첫 글자를 제외한 다른 모든 글자를 * 로 치환할 것이다. 

# 조금 더 편하게 해볼까?

생각한 대로 잘 동작은 하는데 매 필드마다 `JsonSerialize` 를 일일이 다 명시해 줘야 하는 건 좀 불편해 보인다. 

Jackson 라이브러리는 어떤 `JsonSerialize` 을 어떻게 찾아서 어떤 serializer로 처리해야 하는지 알까? 그 부분을 조금 확장하면 `JsonSerialize` 없이도 충분히 가능할 것 같다. 

`JacksonAnnotationIntrospector` 은 어떤 annotation을 어떤 방식으로 처리해야하는지 정의되어있는 클래스고 우리는 serializer를 찾는 방식만 조금 변경해주면 될 것 같다. 
```kotlin
    val objectMapper: ObjectMapper by lazy {
        val mapper = ObjectMapper()
        mapper.registerModule(KotlinModule())
        mapper.setAnnotationIntrospector(CustomAnnotationIntrospector())
        mapper
    }
    
    class CustomAnnotationIntrospector : JacksonAnnotationIntrospector() {
        override fun findSerializer(a: Annotated): Any? {
            val ann = a.getAnnotation(MaskRequired::class.java)
            if (ann != null) {
                val serializer = ann.serializer.using
                return serializer.java
            }
            return super.findSerializer(a)
        }
    }
```
이렇게 `MaskRequired` 만 있어도 정의된 serializer 를 찾을 수 있게 적용해 주자.
```kotlin
    class UserTest {
        object NameMasker : Masking {
            override operator fun invoke(s: String): String = Regex("(?<=.{1}).*").replace(s, "*")
        }
    
        data class User(
            @MaskRequired(masker = NameMasker::class)
            val name: String = ""
        )
    
        @Test
        fun `simple name masking`() {
            val user = User("홍길동")
            val jsonStr = objectMapper.writeValueAsString(user)
            assertEquals("""{"name":"홍**"}""".trimIndent(), jsonStr)
        }
    }
```
다시 테스트 코드를 수정해서 테스트 해보자. 성공!

이렇게 jackson 을 적절히 확장해서 우리가 annotaion 에 정의한 마스킹 함수로 유연하게 민감한 개인정보를 잘 가릴 수 있게 되었다. 

[https://github.com/rutesun/workspace/tree/master/marshaller](https://github.com/rutesun/workspace/tree/master/marshaller)
