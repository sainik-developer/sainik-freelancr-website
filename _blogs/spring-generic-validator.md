---
title: "Generic inter-field bean validator using SpEl"
---

## What is it ? 
It's a generic validator by using spring exprssion language to specify contraints in Spring code. Improves readability by keeping validation logic in 
same POJO/DTO . It's a type(Class) level validator where inter field constraints can be specified easily by SpEl. 

Known benefits:  
1. Avoid writting custom validator for diffrent beans.
2. High readability by keeping constraints in same bean class.
3. Allow to express validation logic by SpEl (Spring expression language)

## Technical background 
### Brief hisroty of validator in Java world

J2EE(Spring) is well familiar with POJO/DTO validator for long, it's well organized, standadised by java community 
and used by most. It's JSR(Java Specification Requests)-380 and implemented and maintained by hibernate community.
Spring supports JSR-380 inherently.

#### What's addressed by JSR-380
JSR-380 requirements is captured in below artifact
```html
<dependency>
    <groupId>javax.validation</groupId>
    <artifactId>validation-api</artifactId>
</dependency>
```
It came up with an extendiable frameowkr which allows user to implement their requirements when default validators does not support so user can extends it.
These are validators specified in `javax.validator.validation-api`
|                 |                   |               |                |
|---------------- | ----------------- | ------------- | -------------- |
| AssertFalse     | Future            | NotBlank      | Pattern        |
| AssertTrue      | FutureOrPresent   | NotEmpty      | Positive       |
| DecimalMax      | Max               | NotNull       | PositiveOrZero |
| DecimalMin      | Min               | Null          | Size           |
| Digits          | Negative          | Past          |                | 
| Email           | NegativeOrZero    | PastOrPresent |                | 

It's implemented by hibernate community and being used by spring

```html
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
</dependency>
```

first contribution 

Nature of all validator is either at class level or field level but does not address the inter field validator in a bean

Second contribution 
Framework to extend the validation for developer using given interfaces and classes in `javax.validator`. 

### What is tried to bring 
#### What is cross field validation ? 
Below is a sample class which as few field. But suppose there is a requirements as below 
Validation requirements 
1. `name` should be not empty string. 
2. `surname` should be non-empty for __PERSON__ and empty for __ORGANIZATION__ customerType.
3. `dob` should be non-empty for __PERSON__ customerType but for __ORGANIZATION__ should be empty.
4. `doi` should be empty for __PERSON__ but non-empty for __ORGANIZATION__ customerType.
5. `addresses` should be size of one for __ORGANIZATION__ and can be two but at least one for __PERSON__ customerType.
6. `customerType` should not be empty and one of the value of enum 
[CustomerType.java](https://github.com/sainik-developer/SpEl-cross-field-validator/blob/main/src/main/java/com/sf/customvalidator/constant/CustomerType.java)

Below a try to impose those validation using `javax.validator`. Now we can see main issue is the conditional validation requirements which here mainly 
depends on `customerType`.
```java
public enum CustomerType {
    PERSON, ORGANIZATION
}
```
```java
public class CustomerDTO {
    @NotEmpty(groups = PostMapping.class)
    private String name;
    @NotEmpty(groups = PostMapping.class)
    private String surname;
    @NotEmpty(groups = PostMapping.class)
    private LocalDate dob;
    @NotEmpty(groups = PostMapping.class)
    private LocalDate doi;
    @Size(min=1, max=2)
    private List<@Valid AddressDTO> addresses;
    @NotEmpty(groups = PostMapping.class)
    private CustomerType customerType;
}
```
It's not possible to enforce the restriction using default set of validator hence developer will opt for custom validator for `CustomerDTO` or write logic in service layer.
Below is the way explained how it can done by custom validator.
```java
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = { CustomerValidatorImpl.class })
@Documented
public @interface CustomerValidator {
    String message() default "failed!";
    Class<?>[] groups() default { };
    Class<? extends Payload>[]payload() default {};
    SpElCrossFieldCondition[] conditions();
}
``` 
```java
public class CustomerValidatorImpl implements ConstraintValidator<CrossFieldValidator, CustomerDTO> {
 @Override
 public boolean isValid(CustomerDTO customerDTO, ConstraintValidatorContext context) {
     return (custmerDTO.getCustomerType() == ORGANIZATION  
                && StringUtils.isEmpty(custmerDTO.getSurname()) 
                && Objects.isNull(custmerDTO.getDOB()) 
                && Objects.nonNull(custmerDTO.getDOI()) 
                && Objects.nonNull(addresses) && addresses.size() == 1)
                || (custmerDTO.getCustomerType() == PERSON 
                && !StringUtils.isEmpty(custmerDTO.getSurname()) 
                && Objects.isNull(custmerDTO.getDOI()) 
                && Objects.nonNull(custmerDTO.getDOB())
                && Objects.nonNull(addresses) && addresses.size() >= 1 && addresses.size() <= 2);
 }
}
``` 

But what is there is conditional cross-field validation required for `AddressDTO` nested object. In that case is current approach we have to write an another custom validator 
related to AddressDTO. 

#### What is sorted here to address above issue ? 
What if we just can declare our cross-field conditional restriction using annotation at POJO/DTO class as below

```java
@Data
@CrossFieldValidator(groups = {PostMapping.class}, conditions = {
 @SpElCrossFieldCondition(IF = "customerType == T(com.sf.customvalidator.constant.CustomerType).ORGANIZATION", 
                            THEN = "surname==null"),
 @SpElCrossFieldCondition(IF = "customerType == T(com.sf.customvalidator.constant.CustomerType).ORGANIZATION", 
                            THEN = "dob == null AND doi != null"),
 @SpElCrossFieldCondition(IF = "customerType == T(com.sf.customvalidator.constant.CustomerType).ORGANIZATION", 
                            THEN = "addresses!=null AND addresses.size() == 1"), 
 @SpElCrossFieldCondition(IF = "customerType == T(com.sf.customvalidator.constant.CustomerType).PERSON", 
                            THEN = "surname!=null AND !surname.isEmpty()"),
 @SpElCrossFieldCondition(IF = "customerType == T(com.sf.customvalidator.constant.CustomerType).PERSON", 
                            THEN = "dob != null AND doi == null"),
 @SpElCrossFieldCondition(IF = "customerType == T(com.sf.customvalidator.constant.CustomerType).PERSON", 
                            THEN = "addresses!=null AND addresses.size() >= 1 AND addresses.size() <= 2")
})
public class CustomerDTO {
    @NotEmpty(groups = PostMapping.class)
    private String name;
    private String surname;
    private LocalDate dob;
    private LocalDate doi;
    private List<@Valid AddressDTO> addresses;
    @NotEmpty(groups = PostMapping.class)
    private CustomerType customerType;
}
```

```java
@Data
@CrossFieldValidator(groups = {PostMapping.class}, conditions = {
        @SpElCrossFieldCondition(IF = "country == T(com.sf.customvalidator.constant.Country).US OR country == T(com.sf.customvalidator.constant.Country).DE", THEN = "areaCode != null && areaCode.length == 5"),
        @SpElCrossFieldCondition(IF = "country == T(com.sf.customvalidator.constant.Country).IND", THEN = "areaCode != null && areaCode.length == 6")
})
public class AddressDTO {
    @NotEmpty(groups = {PostMapping.class})
    private String houseNo;
    @NotEmpty(groups = {PostMapping.class})
    private String lane;
    @NotEmpty(groups = {PostMapping.class})
    private Country country;
    @NotEmpty(groups = {PostMapping.class})
    private String areaCode;
    @NotEmpty(groups = {PostMapping.class})
    private String state;
}
```
What we just achieve by above annotation approach 

1. generic code so no need to write logic using verbose language and framework syntax 
2. higher code readability as validation is written upfront on DTO 

#### How to write condition for the validation ? 
We have to use our familiar spring expression language (SpEl) to expression our `IF` and `THEN` condition, Documentation of spring expression langauge can be found in 
[SpEl](https://docs.spring.io/spring-framework/docs/3.0.x/reference/expressions.html) 

#### Implementation of generic validater is here

[SpEl Cross field validator](https://github.com/sainik-developer/SpEl-cross-field-validator)
you can have a look at implemenation I am here to  talk about how it should be used rather internal details as those are not very interesting. 
[CrossFieldValidatorImpl.java](https://github.com/sainik-developer/SpEl-cross-field-validator/blob/main/src/main/java/com/sf/customvalidator/validator/CrossFieldValidatorImpl.java)
 
 