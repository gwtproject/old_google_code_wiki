# Introduction

GWT supports a subset of JSR 303 Bean Validation.
At compile time GWT validation uses the same Validation Provider you use on the server to create a Validator for all your objects

# Quick Start

Annotate you beans with contstrants
(see [Person.java](http://code.google.com/p/google-web-toolkit/source/browse/trunk/samples/validation/src/main/java/com/google/gwt/sample/validation/shared/Person.java))

```
public class Person {
  @Size(min = 4)
  private String name;
```

Use the standard validation bootstrap to get a Validator on the client and validate your object
(see [ValidationView.java](http://code.google.com/p/google-web-toolkit/source/browse/trunk/samples/validation/src/main/java/com/google/gwt/sample/validation/client/ValidationView.java))
```
Validator validator = Validation.buildDefaultValidatorFactory().getValidator();
Set<ConstraintViolation<Person>> violations = validator.validate(person);
```


Follow this pattern to create a Validator for the objects you want to validate on the client. (see [SampleValidatorFactory.java](http://code.google.com/p/google-web-toolkit/source/browse/trunk/samples/validation/src/main/java/com/google/gwt/sample/validation/client/SampleValidatorFactory.java))

```
public final class SampleValidatorFactory extends AbstractGwtValidatorFactory {

  /**
   * Validator marker for the Validation Sample project. Only the classes listed
   * in the {@link GwtValidation} annotation can be validated.
   */
  @GwtValidation(value = Person.class,
      groups = {Default.class, ClientGroup.class})
  public interface GwtValidator extends Validator {
  }

  @Override
  public AbstractGwtValidator createValidator() {
    return GWT.create(GwtValidator.class);
  }
}
```

Include the module for your Validation Provider.
Add _replace-with_ tag in your gwt modle file telling GWT to use the Validator you just defined
(see [Validation.gwt.xml](http://code.google.com/p/google-web-toolkit/source/browse/trunk/samples/validation/src/main/java/com/google/gwt/sample/validation/Validation.gwt.xml))

```
<inherits name="org.hibernate.validator.HibernateValidator" />
<replace-with
    class="com.google.gwt.sample.validation.client.SampleValidatorFactory">
    <when-type-is class="javax.validation.ValidatorFactory" />
</replace-with>
```

# Best Practices
  * Use Groups to specify what constraints to run on the client. Groups which should not run on the client should be omitted from the `@GwtValidation` annotation to avoid compilation issues.
```
@ServerConstraint(groups = ServerGroup.class)
public class Person {
  @NotNull
  private String name;
}

Validator validator = Validation.buildDefaultValidatorFactory().getValidator();
// validate on the client
Set<ConstraintViolation<Person>> violations = validator.validate(person, Default.class, ClientGroup.class);
if (!violations.isEmpty()) {
  // client-side violation
} else {
  // client-side validations passed so check server-side ones
  greetingService.serverSideValidate(person, new AsyncCallback<SafeHtml>() {
    @Override
    public void onFailure(Throwable caught) {
      if (caught instanceof ConstraintViolationException) {
        // server-side violation
      }
      // some other issue
    }
    @Override
    public void onSuccess(SafeHtml result) {
      // server-side validations passed
    }
  }
}
```

# Unsupported Features
  * XML configuration
  * Validating non GWT compatible classes like Calendar on the client side
  * ConstraintValidatorFactory. `GWT.create()` is used to create ConstraintValidator objects instead.
  * Validation providers other than the default provider. Always use `Validation.buildDefaultValidatorFactory()` or `Validation.byDefaultProvider()` and not `Validation.byProvider()`.
  * Validation provider resolvers. Because there is only one provider the use of provider resolvers is unnecessary.

# Known Issues
[Open issues labeled jsr303](http://code.google.com/p/google-web-toolkit/issues/list?can=2&q=label%3Ajsr303)

## Critical Bugs
  * Message Interpolation can recurse forever.

## Other Issues
  * Many errors that would happen at runtime instead fail to compile.
  * ReportAsSingleViolation has no effect.
  * Message Localization [Issue 5763](http://code.google.com/p/google-web-toolkit/issues/detail?id=5763)