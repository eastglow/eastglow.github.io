
---
layout: post
title:  "[Java]Validation과 Custom Validator"
date:   2022-06-13
author: EastGlow
categories: Back-end
---

개발하다보면 컨트롤러로 들어오는 파라미터에 대한 검증이 필요한 경우가 있다. 원시적인 방법으로는 컨트롤러에서 직접 해당 파라미터에 대한 검증을 마칠 수 있을 것이고, 조금 더 발전된 방법으로는 별도의 Validation을 담당하는 유틸이나 클래스를 만들어서 파라미터만 넘겨주면 검증을 마친 뒤 true or false로 검증 여부를 전달하거나 유틸 안에서 Custom Exception을 발생시켜서 호출 자체를 더이상 진행하지 못하도록 할 수 있을 것이다.

하지만 이러한 방법들은

1) 컨트롤러에서 무언가 코드를 추가로 적어야 한다는 점과
2) 해당 파라미터를 사용하는 영역이 여러 군데라면 똑같은 검증 행위를 여러 번 반복해야할 수 있다는 불편한 점

이 있을 수 있다.

그래서 이러한 부분들을 해결하기 위해 보통 @Valid 어노테이션을 이용하여 손쉽게 파라미터에 대한 검증을 진행하곤 한다. @NotBlank, @Min, @Email 등 이미 구현되어 있는 Validation 관련 어노테이션들이 이용할 수 있다.

이미 이것들에 대한 자료는 차고 넘치기 때문에 기본적으로 구현되어 있는 어노테이션 외에 사용자가 직접 특정 케이스에 대한 파라미터 검증 로직을 구현할 수 있는 방법과 하나의 Validator 안에서 케이스에 따라 각각 다른 메시지를 보여줄 수 있는 방법을 살펴보고자 한다.

> 이미 어느정도 @Valid에 대한 정보를 알고 있다는 가정 하에 부분 부분 생략된 설명들이 있을 수 있으니 양해 바란다.

# 1. Custom Valid Annotation 생성

    @Documented  
    @Constraint(validatedBy = TestReqDtoValidator.class)  
    @Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})  
    @Retention(RetentionPolicy.RUNTIME)  
    public @interface TestReqDtoValid {  

      String message() default "";  

      Class<?>[] groups() default {};  

      Class<? extends Payload>[] payload() default {};  

    }

Validator를 통해 파라미터를 검증할 클래스에 붙일 어노테이션을 정의한다. 클래스에 붙일 수도 있을 것이고 필드에 붙일 수도 있을 것이고 어떤 상황에서 어떤 파라미터를 검증하느냐에 따라 달라질 수 있다.

# 2. Custom Validator 생성

    public class TestReqDtoValidator implements ConstraintValidator<TestReqDtoValid, TestReqDto> {  

      @Autowired  
      private MessageSourceAccessor msa;  

      @Override  
      public boolean isValid(TestReqDto testReqDto, ConstraintValidatorContext context) {  

        try {  

          String testDate = testReqDto.getTestDate();  

          String yyyymmddRegex = "^\\d{4}\\d{2}\\d{2}$";  
          Pattern yyyymmddPattern = Pattern.compile(yyyymmddRegex);  

          if (StringUtils.isNotBlank(testDate) && !yyyymmddPattern.matcher(testDate).matches()) {  
            this.setMessage(msa.getMessage("test.param.invalid"), context);  
            return false;
          }

          return true;

        } catch (NullPointerException e) {  
          log.error("TestReqDtoValidator NPE", e);  
          return false;
        }
        
        private void setMessage(String message, ConstraintValidatorContext context) {  
          context.disableDefaultConstraintViolation();  
          context.buildConstraintViolationWithTemplate(message).addConstraintViolation();  
        }  
    }

1번에서 만든 어노테이션이 붙은 부분에서 실제로 요청받은 파라미터를 받아서 검증하는 Validator 클래스이다.

위 샘플코드에서는 간단하게 testDate라는 yyyymmdd 형태의 String 변수를 받아서 그것이 실제로 숫자로 이루어진 yyyymmdd 형태인지 정규식으로 체크하는 검증을 해주고 있다.

기본적으로 testDate가 있을 때만 Pattern을 통해 정규식 체크를 해주도록 해두었고, 만약 정규식을 통과하지 못했다면 별도로 정의해둔 메시지(test.param.invalid)를 MessageSourceAccessor의 getMessage를 이용하여 meesage.properties에 정의한 메시지 내용을 가져오도록 한다.

그리고 나서 Validator 내부에 private 메소드로 정의해둔 setMessage를 호출하여 위에서 구한 메시지 내용과 ConstraintValidatorContext 객체를 넘겨서 미리 정의된 제약조건에 대한 옵션값을 disable 처리하고(disableDefaultConstraintViolation) 새로운 제약조건 템플릿을 앞서 넘겨받은 메시지 내용으로 build(buildConstraintViolationWithTemplate)하여 마지막으로 addConstraintViolation을 호출하여 마무리해준다.

disableDefaultConstraintViolation을 호출하지 않고 바로 buildConstraintViolationWithTemplate를 호출하면 실제로 Validator의 검증을 통과하지 못했을 때 제약조건 템플릿을 두번 사용하게 되면서 실제 검증 오류가 한번 걸렸지만 2개의 오류가 있다고 로그가 찍히게 된다.

disableDefaultconstraintViolation 옵션값에 대한 주석을 살펴보니 아래와 같이 언급되어 있었다.

> Disables the default ConstraintViolation object generation (which is using the message template declared on the constraint).
> Useful to set a different violation message or generate a ConstraintViolation based on a different property.

TestReqDtoValid 어노테이션에서 message 필드의 default값을 설정하는 등 기본 제약조건 템플릿의 설정값을 disableDefaultconstraintViolation 옵션으로 사용하지 않게 처리를 해준 뒤에 별도로 추가한 template를 사용할 수 있다는 의미인 듯 하다.

어쨌든 위 코드처럼 설정한 뒤 @TestReqDtoValid 어노테이션이 붙은 파라미터를 사용하는 컨트롤러를 호출해보면 testDate가 yyyymmdd 형식의 String 변수가 아닐 경우 아래와 같은 로그를 만날 수 있게 된다.

> .m.m.a.ExceptionHandlerExceptionResolver : Resolved [org.springframework.web.bind.MethodArgumentNotValidException: Validation failed for argument [0] in public my.eastglow.comm.dto.ResponseDto<my.eastglow.test.dto.TestResDto> my.eastglow.test.controller.TestController.testMethod(my.eastglow.test.dto.TestReqDto): [Error in object 'testReqDto': codes [TestReqDtoValid.testReqDto,TestReqDtoValid]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [testReqDto.,]; arguments []; default message []]; default message [testDate가 잘못된 형식입니다.]] ]

아, 참고로 나는 위 MethodArgumentNotValidException을 잡는 별도의 ExceptionHandler 클래스를 두고 따로 처리해주고 있다. Exception 발생 시엔 별도로 정의해둔 공통 DTO를 사용하고 있으며 http status 관련 필드와 함께 어떤 오류코드, 오류메시지가 발생하였는지를 필드에 담아서 내려주고 있다. 이처럼 어느정도 예측 가능한 Exception들은 ExceptionHandler를 별도로 두어서 한방에 처리하면 편하고 좋은 것 같다.

# 3. 마무리

이번 글에서 소개한 @Valid 어노테이션에 대한 것은 꽤나 많이 쓰이고 있고 유명하기 때문에 참고할 수 있는 글들도 상당히 많은 편이다. 그래서 상대적으로 글이 없었던 Custom Validator 구성 방법과 함께 하나의 Validator 안에서 케이스에 따라 메시지를 유동적으로 세팅해줄 수 있는 방법에 대해 설명해보았다.

Custom Validator를 별도로 두고 사용하는 이점이 상당히 많기 때문에 개인적으로 파라미터에 대한 검증은 이 방법을 이용하는 편이다. 물론 상황에 따라 서비스 레이어에서 특정 로직에서 검증이 필요한 경우엔 별도의 유틸 메소드를 구성하여 사용하는 편이 편할 수도 있을 것이고, static 메소드를 사용했을 때 더 편하고 장점이 있는 케이스도 있을 것이다. 상황에 따라 적절하게 맞는 방법을 잘 사용하는 것도 중요하며 이런 부분은 많은 코드들을 경험해봐야 몸으로 습득할 수 있다고 생각한다. 나도 단편적인 케이스에 대한 방법을 설명한 것이기 때문에 내가 소개한 검증 방법이 무조건 옳다는 것은 아니니 이 글을 읽는 분들이 잘 판단하여 적절하게 사용하면 좋을 것 같다.
