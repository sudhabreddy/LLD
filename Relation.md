C# has no special keyword for any of these. All three are written as ordinary references between classes. What makes them different is how the objects are created, stored and passed around. Here is how each one is typically implemented.

Association

Definition: One class uses or knows about another, with no ownership. Both objects live independently.

How you write it in C#: The other object is passed in, usually as a method parameter, or held as a reference without any sense of ownership.

csharp
public class Printer
{
    public void Print(Document doc) => Console.WriteLine(doc.Text);
}

public class Document
{
    public string Text { get; set; } = "";
}

// Printer uses Document only while printing. No ownership.
new Printer().Print(new Document { Text = "Hello" });
Aggregation

Definition: A whole-part (“has-a”) relationship where the part can exist without the whole.

How you write it in C#: The part is created outside and passed in through the constructor or a method, and the whole stores it in a field or collection.

csharp
public class Professor
{
    public string Name { get; }
    public Professor(string name) => Name = name;
}

public class Department
{
    private readonly List<Professor> _professors = new();

    // Part comes from outside: the department doesn't create it.
    public void Add(Professor p) => _professors.Add(p);
}

var prof = new Professor("Dr. Iyer");   // exists on its own
var dept = new Department();
dept.Add(prof);                         // aggregated into the department
Composition

Definition: A whole-part relationship where the part cannot exist without the whole. The whole owns the part’s lifetime.

How you write it in C#: The whole creates the part itself, usually in its constructor or its own methods, and doesn’t hand it out for other objects to keep or share.

csharp
public class Room
{
    public string Name { get; }
    internal Room(string name) => Name = name;   // discourages outside creation
}

public class House
{
    private readonly List<Room> _rooms = new();

    public House()
    {
        // The house creates its own rooms.
        _rooms.Add(new Room("Kitchen"));
        _rooms.Add(new Room("Bedroom"));
    }

    public IReadOnlyList<Room> Rooms => _rooms.AsReadOnly();
}
The quick rule
	How the other object arrives	Stored as a field?	Lifetime
Association	Passed in, often as a method parameter	Usually not	Independent
Aggregation	Created outside, passed in	Yes	Independent
Composition	Created inside the whole using new	Yes, kept private	Tied to the whole

One thing worth knowing: some authors treat a stored reference that’s passed in (like constructor injection) as association rather than aggregation. The line between those two is not drawn the same way in every textbook. The line between aggregation and composition is much more consistent: who creates the part, and whether it can live on its own.

give me more example in realtime which clearly states composition vs aggregation with code

The clearest way to see the difference is to look at one real system where both relationships appear side by side. In each example below, the same “whole” class has one part it owns (composition) and one part it only refers to (aggregation).

Example 1: E-commerce order

An order owns its order lines, but it only refers to a customer.

If an order is cancelled and deleted, its lines (“2 × Keyboard”) are meaningless and go with it. That’s composition.
The customer still exists after the order is deleted. They can place other orders. That’s aggregation.
csharp
public class Customer
{
    public string Name { get; }
    public Customer(string name) => Name = name;
}

public class OrderLine
{
    public string Product { get; }
    public int Quantity { get; }
    public decimal Price { get; }

    internal OrderLine(string product, int quantity, decimal price)
    {
        Product = product;
        Quantity = quantity;
        Price = price;
    }
}

public class Order
{
    public Customer Customer { get; }                 // AGGREGATION
    private readonly List<OrderLine> _lines = new();  // COMPOSITION

    // Customer is created elsewhere and passed in.
    public Order(Customer customer) => Customer = customer;

    // Order lines are created inside the order.
    public void AddItem(string product, int qty, decimal price)
        => _lines.Add(new OrderLine(product, qty, price));

    public decimal Total => _lines.Sum(l => l.Quantity * l.Price);
}

// Usage
var sudha = new Customer("Sudha");

var order1 = new Order(sudha);
order1.AddItem("Keyboard", 2, 25m);

var order2 = new Order(sudha);   // same customer, another order
order2.AddItem("Monitor", 1, 150m);

order1 = null;   // order1 and its lines are gone
Console.WriteLine(sudha.Name);   // the customer is still here
Example 2: Banking

A bank account owns its transactions, but refers to the branch it belongs to.

A transaction record only exists as part of an account’s history. That’s composition.
A branch exists before the account and continues after the account is closed. That’s aggregation.
csharp
public class Branch
{
    public string Code { get; }
    public Branch(string code) => Code = code;
}

public class Transaction
{
    public DateTime Date { get; }
    public decimal Amount { get; }

    internal Transaction(decimal amount)
    {
        Date = DateTime.UtcNow;
        Amount = amount;
    }
}

public class BankAccount
{
    public Branch Branch { get; }                                // AGGREGATION
    private readonly List<Transaction> _transactions = new();   // COMPOSITION

    public BankAccount(Branch branch) => Branch = branch;

    public void Deposit(decimal amount)
        => _transactions.Add(new Transaction(amount));

    public void Withdraw(decimal amount)
        => _transactions.Add(new Transaction(-amount));

    public decimal Balance => _transactions.Sum(t => t.Amount);
}

// Usage
var mkBranch = new Branch("MK-001");

var savings = new BankAccount(mkBranch);
savings.Deposit(1000m);
savings.Withdraw(200m);

var current = new BankAccount(mkBranch);   // branch shared by many accounts

The branch is shared across many accounts, which is a strong sign of aggregation. Transactions are never shared between accounts, which is a sign of composition.

Example 3: Online learning platform

A course owns its modules, but refers to its enrolled students.

If a course is removed, its modules (“Module 3: Async/Await”) have no reason to exist. That’s composition.
Students remain on the platform and stay enrolled in other courses. That’s aggregation.
csharp
public class Student
{
    public string Name { get; }
    public Student(string name) => Name = name;
}

public class Module
{
    public string Title { get; }
    internal Module(string title) => Title = title;
}

public class Course
{
    public string Title { get; }
    private readonly List<Module> _modules = new();     // COMPOSITION
    private readonly List<Student> _students = new();   // AGGREGATION

    public Course(string title) => Title = title;

    public void AddModule(string title) => _modules.Add(new Module(title));
    public void Enrol(Student student) => _students.Add(student);
}

// Usage
var arun = new Student("Arun");

var csharp = new Course("C# Fundamentals");
csharp.AddModule("Classes and Objects");
csharp.AddModule("OOP Relationships");
csharp.Enrol(arun);

var kafka = new Course("Kafka Basics");
kafka.Enrol(arun);   // same student in two courses
Example 4: Company HR system

A company owns its departments, but departments only refer to employees.

If the company is dissolved, its departments cease to exist. That’s composition.
If a department is closed, the employees can be moved elsewhere. That’s aggregation.
csharp
public class Employee
{
    public string Name { get; }
    public Employee(string name) => Name = name;
}

public class Department
{
    public string Name { get; }
    private readonly List<Employee> _staff = new();     // AGGREGATION

    internal Department(string name) => Name = name;

    public void Assign(Employee e) => _staff.Add(e);
    public void Release(Employee e) => _staff.Remove(e);
}

public class Company
{
    private readonly List<Department> _departments = new();   // COMPOSITION

    public Department CreateDepartment(string name)
    {
        var dept = new Department(name);
        _departments.Add(dept);
        return dept;
    }

    public void CloseDepartment(Department dept) => _departments.Remove(dept);
}

// Usage
var company = new Company();
var it = company.CreateDepartment("IT");
var hr = company.CreateDepartment("HR");

var priya = new Employee("Priya");
it.Assign(priya);

// IT is closed, but Priya moves to HR.
it.Release(priya);
company.CloseDepartment(it);
hr.Assign(priya);

This example is useful because Department is the part in one relationship (composition with Company) and the whole in another (aggregation with Employee). The same class can play both roles.

Summary across all four examples
System	Composition (owned part)	Aggregation (referenced part)
E-commerce	Order → OrderLine	Order → Customer
Banking	BankAccount → Transaction	BankAccount → Branch
Learning platform	Course → Module	Course → Student
HR	Company → Department	Department → Employee
The code patterns to look for

Composition usually shows up as:

The whole calls new to create the part
The part’s constructor is internal or private
The part is never shared between two wholes

Aggregation usually shows up as:

The part is created outside and passed into a constructor or method
The same part instance can appear in more than one whole
The part can be removed and attached somewhere else

A note on real projects: the same pair of classes can be modelled either way depending on the business rules. For instance, if your bank allowed transactions to be moved between accounts, the Account → Transaction relationship would stop being composition. So the classification comes from what the business says about lifetime and ownership, not from the class names.
