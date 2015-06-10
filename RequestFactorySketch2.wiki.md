# Introduction

Here's the next take on RequestFactory, the proposed system to easy client access to server side ORM entities.

As much as I'd prefer to do this in Wave, ValueStoreAndRequestFactory was just too painful: ExportyBot really sucks, which is a shame. Until we have all of the community waving, I'll stick with the Wiki. (But if anyone wants to reply to this via Wave, I'm in!)

Following is another iteration on the RequestFactory idea. I still don't have an actual design doc for you all, but am working on prototype code.

The idea continues to be that you run a JPA-savvy script and it generates a RequestFactory interface for you, with builder style classes that embody any static finder and other service methods you've defined. That is, when you want to call a server method, you build a request object tailored to it. (My assumption is that other scripts and servlets can be provided for other persistence frameworks.)

Changes this time include the use of a refined, entity aware EventBus, which lets you subscribe to either whole classes of event (e.g., `EmployeeUpdateEvent implements EntityUpdateEvent<Employee>`), or only to those events related to a specific entity.

Responses to server calls do not include any entity fields, just their ids. To get at the meat you subscribe to appropriate events. These events are also called as entities are added, updated, deleted, etc. by other service calls from other parts of the app.

Note that code is written to assume that service calls must always be made. We assume that caching will be in place, and that redundant calls will short circuit. There is no magic to try to make server calls for you, although I do assume that we'll batch them.

RequestFactory no longer ties directly to HasValue and the proposed HasValueList. In fact I'm ignoring UI data binding in this sketch, but still presuming we can automate the bulk of it and that those two interfaces will play a key role.


# Bunch o' Fantasy Code

```
interface Entity {
  Object getId();
  Comparable<?> getVersion();
}

@ServerType(com.google.gwt.sample.valuebox.domain.Employee.class)
public class Employee implements Entity {
  private final String id;
  private final Integer version;

  Employee(String id, Integer version) {
    this.id = id;
    this.version = version;
  }

  public static final Property<Employee, String> USER_NAME =
      new Property<Employee, String>(Employee.class, String.class);
  public static final Property<Employee, String> DISPLAY_NAME =
      new Property<Employee, String>(Employee.class, String.class);
  public static final Property<Employee, Employee> SUPERVISOR =
      new Property<Employee, Employee>(Employee.class, Employee.class);

  public String getId() {
    return id;
  }

  public Integer getVersion() {
    return version;
  }
}

class EmployeeList {
  private final EventBus events;
  private final RequestFactory requests;
  EmployeeList(EventBus events, RequestFactory requests) {
    this.events = events;
    this.requests = requests;
  }

  public void init() {
    events.addHandler(this, EmployeeUpdate.class); // all of them
    requests.employees()
        .findAllEmployees()
        .forCallback(new MyEmployeeRequestCallback()).
        .forProperty(Employee.USER_NAME)
        .forProperty(Employee.DISPLAY_NAME)
        .fire();
  }

  class MyEmployeeRequestCallback implements EntityRequestCallback<Employee> {
    void onSuccess(List<Employee> employees) {
      /* 
       * set the list model ids, but we have no values yet. They
       * will come through calls to onEmployeeUpdate
       */
    }
  }

  /** Called by event bus. Could come before the request is fired! */
  void onEmployeeUpdate(Set<Values<Employee>> values) {
    /* tweak the list model */
  }
}

class EmployeeDetails {
  private final EventBus events;
  private final RequestFactory requests;
  EmployeeDetails(EventBus events, RequestFactory requests) {
    this.events = events;
    this.requests = requests;
  }

  public void init(Employee e) {
    events.addHandler(this, EmployeeUpdate.class, e);
    requests.employees()
        .findEmployee(e)
        .fire(); // Note didn't bother with callback, will get all properties
  }

  /** Called by event bus. Could come before the request is fired! */
  void onEmployeeUpdate(Set<Values<Employee>> values) {
    /* tweak the list model */
  }
}

/**
 * Assumes editors as described in slice of life wave
 * https://wave.google.com/wave/#restored:wave:googlewave.com!w%252B3IMMyKcVB.1
 */
class EmployeeUpdateOrCreate {
  private final RequestFactory requests;
  private final Editor<Employee> editor;
  private DeltaValueStore deltas;
  private boolean creating = false;

  EmployeeUpdateOrCreate(RequestFactory requests, Editor<Employee> editor) {
    this.requests = requests;
    this.editor = editor;
    editor.setListener(this);
    deltas = requests.getValueStore().edit();
    editor.setStore(deltas);
  }

  public void doNew() {
    creating = true;
    Employee newEmployee = deltas.newEntity(Employee.class); // ORLY?
    doUpdate(newEmployee);
  }

  public void doUpdate(Employee e) {
    /* perhaps responsible for exposing the editor */
    editor.setValue(e);
  }

  /** Called by editor */
  public onCancel(Editor<Employee> ignored) {
    if (deltas.isChanged()) { /* warn or something */ }
  }

  /** Called by editor */
  public onSave(Editor<Employee> ignored) {
    if (!deltas.validate()) return;
    requests.lifeCycle(deltas)
        .forOperation(creating ? LifeCycle.CREATE : LifeCycle.DELETE)
        .<Employee> forEntity(editor.getValue())
        .forCallback(new MyLifeCycleCallback())
        .fire();
  }

  class MyLifeCycleCallback() implements LifeCycleCallback<Employee> {
    void onSuccess(Employee e) {
      /* no access to values. Tear down the editor. Interested
       * parties will be notifed via bus */
    }

    void onFailure(Exception e) {
      if (e instanceof ValidationException) {
        // Or should this happen automatically, as in the first sketch?
        editor.setErrors(((ValidationException) e).getErrors);
      }
    }
  }
}

/** Reports have ReportItems */
class ReportDetails {
  private final EventBus events;
  private final RequestFactory requests;
  ReportDetails(EventBus events, RequestFactory requests) {
    this.events = events;
    this.requests = requests;
  }

  public void init(Report r) {
    events.addHandler(this, ReportUpdate.class, e);
    requests.reports()
        .findReport(r)
        .fire(); // To get all properties
    requests.reportItems()
        .forCallback(new MyReportItemCallback())
        .findItemsForReport(r)
        .fire();
  }

  class MyReportItemRequestCallback implements EntityRequestCallback<ReportItem> {
    void onSuccess(List<ReportItem> items) {
      events.addHandler(this, ReportItemUpdate.class, items);
    }
  }

  void onReportUpdate(Set<Values<Report>> values) {
    /* show the values, maybe via ui data binding */
  }

  void onReportItemUpdate(Set<Values<ReportItem>> values) {
    /* tweak the list model for the report items */
  }
}
```