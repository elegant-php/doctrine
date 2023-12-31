# Elegant PHP objects doctrine

PHP implementation of [Elegant Objects](https://www.amazon.com/dp/B09R21DKSC?binding=paperback).

This is a theory. All code examples are abstract.

## Table of contents

- [Elegant PHP objects doctrine](#elegant-php-objects-doctrine)
  - [Table of contents](#table-of-contents)
  - [Sizes](#sizes)
  - [Ecosystem](#ecosystem)
    - [Always use interfaces](#always-use-interfaces)
    - [Composition over inheritance](#composition-over-inheritance)
    - [Tests](#tests)
  - [Low coupling, high cohesion](#low-coupling-high-cohesion)
    - [Temporal coupling](#temporal-coupling)
    - [Use vertical decomposition of responsibility](#use-vertical-decomposition-of-responsibility)
  - [Naming](#naming)
    - [Classes](#classes)
    - [Methods](#methods)
    - [Objects and variables](#objects-and-variables)
  - [Creation of objects and their state](#creation-of-objects-and-their-state)
    - [Build immutable objects only](#build-immutable-objects-only)
    - [Multiple constructors](#multiple-constructors)
    - [Keep constructors code-free](#keep-constructors-code-free)
    - [Encapsulate as little as possible](#encapsulate-as-little-as-possible)
    - [Encapsulate something](#encapsulate-something)
    - [Identification](#identification)
  - [Evils](#evils)
    - [Don't use globals](#dont-use-globals)
    - [Don't use NULL](#dont-use-null)
    - [Don't use static methods](#dont-use-static-methods)
    - [Don't use getters and setters](#dont-use-getters-and-setters)
    - [No reflection](#no-reflection)
    - [Don't use private constants](#dont-use-private-constants)
    - [Don't configure objects](#dont-configure-objects)
    - [Don't use annotations](#dont-use-annotations)
  - [Exceptions](#exceptions)
    - [Fail first instead of fail safe](#fail-first-instead-of-fail-safe)
    - [Declare exceptions](#declare-exceptions)
    - [Don't catch exceptions unless you have to](#dont-catch-exceptions-unless-you-have-to)
    - [Chain exceptions everywhere](#chain-exceptions-everywhere)
    - [Recover only at top entry points](#recover-only-at-top-entry-points)
  - [Patterns and principles](#patterns-and-principles)
    - [RAII](#raii)
    - [Factories](#factories)
    - [Law of Demeter](#law-of-demeter)
    - [AOP for technical moments](#aop-for-technical-moments)
    - [Validation](#validation)

## Sizes

Try to stay under **250** lines and lower.

Try to create **5** or less number of public methods.

Use small interfaces, one object can implement a number of them.

Objects must be very cohesive (their methods and properties should be close to each other).

Try to encapsulate [4](#encapsulate-as-little-as-possible) or less objects in one object.

**[Table of contents ↑](#table-of-contents)**

## Ecosystem

### Always use interfaces

Make sure that all public methods in your class is an implementation of an interface.

Examples:

- *Bad*

```php
interface Foo
{
    public function bar(): void;
}

class DefaultFoo implements Foo
{
    public function bar(): void {}

    public function baz(): void {}
}
```

- *Better*

```php
interface Bar
{
    public function bar(): void;
}

interface Baz
{
    public function baz(): void;
}

final class Foo implements Bar, Baz
{
    public function bar(): void {}

    public function baz(): void {}
}
```

**[Table of contents ↑](#table-of-contents)**

### Composition over inheritance

Make classes final or abstract. Make abstract class methods final (don't override abstract class methods).

Extend types(interfaces and abstract classes) only.

Examples:

- *Bad*

```php
class Foo
{
    public function baz()
    {
        // foo implementation
    }

    public function quux()
    {
        // foo implementation
    }
}

class Bar extends Foo
{
    public function quux()
    {
        // bar implementation
    }
}
```

- *Better*

```php
interface Baz
{
    public function baz(): void;
}

interface Quux extends Baz
{
    public function quux(): void;
}

interface Foo extends Baz, Quux
{
}

final class DefaultFoo implements Foo
{
    public function baz()
    {
        // foo implementation
    }

    public function quux()
    {
        // foo implementation
    }
}

final class DefaultBar implements Bar, Baz
{
    public function __construct(
        private readonly Foo $foo
    ) { }

    public function baz()
    {
        $this->foo->baz();
    }

    public function quux()
    {
        // bar implementation
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Tests

Each class should have a unit test.  
Unit test names should shows what objects really do.

Each interface should have a fake object.  
Use fake objects instead of mock and stub objects.

Examples:

- *Bad*

```php
class SaveCommentTest
{
    public function testWithValidUser()
    {
        $userRepository = $this->createMock(UserRepository::class);
        $userRepository
            ->method('getById')
            ->willReturn(new User(/* some data */));

        arrest_that($this->saveCommentService->save($userRepository, 'message'));
    }
}
```

- *Better*

```php
interface PgSqlUser
{
    public function userById(IntegerType $id): User;
}

class PgSqlFakeUser implements PgSqlUser
{
    public function userById(IntegerType $id): User
    {
        return $this->user;
    }
}

final class PutCommentActionTest implements TestCase
{
    public function susscessful(): void
    {
        assert_that(
            (new PutCommentAction(new PgSqlFakeUser(), 'message'))->succsessful()
        );

        // the same with temporal coupling:
        //
        // $user = new PgSqlFakeUser();
        // $action = new PutCommentAction($user, 'message');
        //
        // arrest_that($action->succsessful());
    }

    public function failed(): void
    {
        assert_not(
            (new PutCommentAction(new PgSqlFakeUser()))->succsessful()
        );
    }
}
```

**[Table of contents ↑](#table-of-contents)**

## Low coupling, high cohesion

### Temporal coupling

Try to write code where each line is independent, otherwise replacing the lines will cause a syntax error.

Examples:

- *Bad*

```php
final class DefaultFoo implements Foo
{
    public function process(Dependency $dependency): void
    {
        // if we swap the return statements below, the behavior will be different without an error
        // it is a hidden coupling, the first "return" depends on the "if" condition

        if ($dependency->someExpected()) {
            return $this->bar();
        }

        return $this->baz();
    }
}
```

- *Better*

```php
final class BarFoo implements Foo
{
    public function process()
    {
        // bar
    }
}

final class BazFoo implements Foo
{
    public function process()
    {
        // baz
    }
}

$foo = new ObjectiveIf(
    condition: $dependency,
    then: new BarFoo,
    else: new BazFoo,
);
```

**[Table of contents ↑](#table-of-contents)**

### Use vertical decomposition of responsibility

Try to use vertical instead of horizontal decomposition of responsibility.

Examples:

- *Bad*

```php
final class FileLog implements Log
{
    public function __construct(
        private readonly string $path
    ) {}

    public function put(Message $message): void
    {
        file_put_contents($this->path, $message->value());
    }
}

final class TimedMessage implements Message, StringType
{
    public function __construct(
        private readonly string $message
    ) {}

    public function value(): string
    {
        return sprintf("%s: %s", date('Y-m-d'), $this->message);
    }
}

// the script below knows both Log and Message interfaces, it's a horizontal decomposition of responsibility

$log = new FileLog('/path/to/log');
$message = new TimedMessage('message');

$log->put($message);
```

- *Better*

```php
final class TimedLog implements Log
{
    public function __construct(
        private readonly Log $log
    ) {}

    public function put(string $message): void
    {
        $this->log->put(sprintf("%s: %s", date('Y-m-d'), $message));
    }
}

// now we only use the Log interface, it's a vertical decomposition of responsibility

$log = new TimedLog(new FileLog('/path/to/log'));
$log->put('some message');
```

**[Table of contents ↑](#table-of-contents)**

## Naming

### Classes

> Class names must be named by what them objects are, not what they do.  

Use nouns and avoid postfix "-er".

Examples:

- *Bad*

```php
class DataProvider
{
    public function __construct(
        private readonly QueryBuilder $queryBuilder
    ) { }

    public function getData(string $table, array $select, array $where): array
    {
        $query = $this
            ->queryBuilder
            ->select($select)
            ->from($table)
            ->where($where);

        return $this->fetchAll();
    }
}
```

- *Better*

```php
final class FooSQLReport implements FooReport
{
    public function __construct(
        private readonly DbConnection $connection,
        private readonly TableName $tableName
    ) { }

    public function asArrayBetweenDates(DateTime $from, DateTime $to): ArrayType
    {
        return $this->connection->sqlResultAsArray(
            sql: 'SELECT * FROM ?s ...',
            tableName: this->tableName->value(),
        );
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Methods

Use nouns when a method will create something and return it.  
Use verbs when a method manipulates.  
Use adjectives when a method returns a boolean value (or make those values objects and use nouns).

Never mix builders and manipulators together.

Examples:

- *Bad*

```php
class File
{
    public function save(string $path): bool
    {
        $succsessful = file_put_contents($path, $this->file);

        return (bool)$successful;
    }
}
```

- *Better*

```php
final class FileSavingResult implements Result
{
    public function __construct(
        private readonly bool $successful = false
    ) { }

    public function from(bool $result): static
    {
        return new static($result);
    }

    public function successful()
    {
        return $this->successful;
    }
}

final class MyFile implements File
{
    public function __construct(
        private readonly ResourceFile $file,
        private readonly Result $result
    ) {}

    public function savedTo(StringType $path): static
    {
        return new static(
            file: $this->file,
            result: $this->result->from(file_put_contents($path, $this->file))
        );
    }

    public function successful(): bool
    {
        return $this->result->successful();
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Objects and variables

Use single and plural nouns for variable names, or refactor the code.

Use a noun with an adjective if the variable losses its meanings without the adjective (for example: 'timeZone', 'microService').

Examples:

- *Bad*

```php
class DaysHelper
{
    public function getWorkingDays()
    {
        $allDays = $this->repository->getDays();

        $workingDays = [];
        foreach ($allDays as $day) {
            if ($day->isWorking()) {
                $workingDays[] = $day;
            }
        }

        return $workingDays;
    }
}
```

- *Better*

```php
final class WorkingDays implements Days
{
    public function __construct(
        private readonly AllDays $period,
        private Collection $collection
    ) { }

    public function asArray(): array
    {
        // it's better to create an objective mapping: new Map(...)
        foreach ($this->period as $day) {
            if (!$day->working()) {
                continue;
            }

            $this->collection->add($day);
        }

        return $this->collection->asArray();
    }
}
```

**[Table of contents ↑](#table-of-contents)**

## Creation of objects and their state

### Build immutable objects only

Objects must be immutable, but they can represent mutable data, such as a file on disk.

Examples:

- *Bad*

```php
class Foo
{
    private string $bar;

    public function setBar(string $bar)
    {
        $this->bar = $bar;
    }
}
```

- *Better*

```php
final class DefaultFoo implements Foo
{
    public function __construct(
        private readonly StringType $bar
    ) {}
}
```

**[Table of contents ↑](#table-of-contents)**

### Multiple constructors

Object creation should be flexible. Try to keep number of object creation ways greater then number of object methods.

Emulate a "primary constructor" after others.

Examples:

- *Bad*

```php
class Foo
{
    public function __construct()
    {
        // empty
    }

    public function createFromInt(int $input)
    {
        // Foo creation
    }

    public function createFromString(string $input)
    {
        // Foo creation
    }
}
```

- *Better*

```php
final class DefaultFoo implements Foo
{
    private SomeType $data;

    /**
     * @param array<string,StringType> type
     *
     * @throws TypeException
     */
    public function __construct(array $type) // you can design it better
    {
        if (array_key_exists('string', $type)) {
            $this->data = new SomeTypeFromString($type['string']);
        } else if (array_key_exists('int', $type)) {
            $this->data = new SomeTypeFromInt($type['int']);
        }

        // primary constructor below

        throw new TypeException('undefined type');
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Keep constructors code-free

Constructors must encapsulate data and must not manipulate it.

Examples:

- *Bad*

```php
class Foo
{
    private int $value;

    public function __construct(string $inputValue)
    {
        $this->value = (int)$inputValue; // type conversion behavior
    }
}
```

- *Better*

```php
interface IntegerType
{
    public function value();
}

final class StringAsInteger implements IntegerType
{
    public function __construct(
        private readonly StringType $value
    ) {}

    public function value()
    {
        return (int)$this->value;
    }
}

final class DefaultFoo implements Foo
{
    private readonly IntegerType $value;

    public function __construct(StringType $value)
    {
        $this->value = new StringAsInteger($value);
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Encapsulate as little as possible

Try to encapsulate 4 or less objects in one object.

Examples:

- *Bad*

```php
class User
{
    private int $id;
    private string $firstName;
    private string $lastName;
    private string $country;
    private string $city;
    private string $street;
    private bool $isAdmin;
    private bool $canViewSomeReport;
}
```

- *Better*

```php
final class AuthorizedUser implements User
{
    private readonly PersonalInformation $personalInformation;
    private readonly Address $address;
    private readonly Role $role;
}
```

**[Table of contents ↑](#table-of-contents)**

### Encapsulate something

Objects must not have an empty body.

Examples:

- *Bad*

```php
class Year
{
    public function current()
    {
        $dateTime = new DateTime();

        return $dateTime->format('Y');
    }
}
```

- *Better*

```php
final class Year implements StringType
{
    public function __construct(
        private DateTime $date
    ) {}

    public function value(): string
    {
        return $this->date->format('Y');
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Identification

The state of objects must be their identity.

Examples:

- *Bad*

```php
class Foo
{
    public function __construct(
        private readonly SomeString $string
    ) { }
}

$a = new Foo('123');
$b = new Foo('123');

if ($a === $b) { // will be false
    // do something
}
```

- *Better*

```php
final class DefaultFoo implements Foo
{
    public function __construct(
        private readonly SomeString $string
    ) { }
}

$a = new Foo('123');
$b = new Foo('123');

if ($a->equals($b)) { // should be true
    // do something
}
```

**[Table of contents ↑](#table-of-contents)**

## Evils

### Don't use globals

Don't use public static properties and constants, global variables e.t.c.

Examples:

- *Bad*

```php
class Logger
{
    public function logWarning(string $message)
    {
        $this->loggerService->warning($message . PHP_EOL); // global constant PHP_EOL
    }
}
```

- *Better*

```php
final class EOLString implements StringType
{
    public function __construct(
        private readonly StringType $string
    ) {}

    public function value(): StringType
    {
        return $this->string . PHP_EOL; // use this legacy in only one place
    }
}

final class WarningLog implements Log
{
    public function __construct(
        private readonly LoggingService $service,
        private readonly EOLString $message
    ) {}

    public function log(): void
    {
        $this->service->warning($this->message);
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Don't use NULL

Act as if NULL did not exist.

Examples:

- *Bad*

```php
class User
{
    private string $sessionId;
    private ?string $name = null;

    public function authorize($name)
    {
        if ($name === null) {
            throw new Exception();
        }

        $this->name = $name;
    }
}
```

- *Better*

```php
final class Guest implements User
{
    public function __construct(
        private StringType $sessionId
    ) {}
}

final class AuthorizedUser implements User
{
    public function __construct(
        private StringType $sessionId,
        private StringType $name
    ) {}
}
```

**[Table of contents ↑](#table-of-contents)**

### Don't use static methods

Examples:

- *Bad*

```php
class User
{
    private User $instance;
    private function __construct() {}

    public static function getInstance(): User
    {
        if ($this->instance === null) {
            $this->instance = new static();
        }

        return $this->instance;
    }
}
```

- *Better*

```php
final class AuthorizedUser implements User
{
    // encapsulation
}

final class ReportPage implements Page
{
    public function __construct(
        private readonly User $user,
        private readonly ReportPageTemplate $template,
    ) { }

    public function html(): Html
    {
        return $this->template->parsedTemplate($this->user);
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Don't use getters and setters

Examples:

- *Bad*

```php
class User
{
    private string $name;
    private string $phone;
    private string $address;

    public function getName(): string
    {
        return $this->name;
    }

    public function setName(string $name): static
    {
        $this->name = $name;

        return $this;
    }

    public function getPhone(): string
    {
        return $this->phone;
    }

    public function setPhone(string $phone): static
    {
        $this->phone = $phone;

        return $this;
    }

    public function getAddress(): string
    {
        return $this->address;
    }

    public function setAddress(string $address): static
    {
        $this->address = $address;

        return $this;
    }
}
```

- *Better*

```php
final class AuthorizedUser implements User
{
    public function __construct(
        private UserName $name,
        private UserPhone $phone,
        private UserAddress $address
    ) {}

    public function changePhone(UserPhone $phone): static
    {
        return new static($this->name, $phone, $this->address);
    }

    public function address(): UserAddress
    {
        return $this->address;
    }

    public function printWith(Template $template): void
    {
        return $template
            ->with('name', $this->name)
            ->with('phone', $this->phone)
            ->with('address', $this->address)
            ->print();
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### No reflection

Avoid any reflections.

Examples:

- *Bad*

```php
class Foo
{
    public function getQuuxByObject($object): Quux
    {
        if ($object instanceof Bar || $object::class === '\Baz') {
            return $this->secondQuux();
        }

        return $this->firstQuux();
    }
}
```

- *Better*

```php
interface Foo
{
    public function quuxForBar(): Quux;

    public function quuxForBaz(): Quux;
}
```

**[Table of contents ↑](#table-of-contents)**

### Don't use private constants

Don't use private constants and static variables, use encapsulation or dependency injection or uniqueness.

Examples:

- *Bad*

```php
final class DefaultOrder implements Order
{
    private static $errorMessage = 'something wrong';

    public function sell(): void
    {
        if (true) { // something wrong
            throw new Exception(static::$errotMessage);
        }

        // sell
    }

    public function decline(): void
    {
        if (true) { // something wrong
            throw new Exception(static::$errotMessage);
        }

        // decline
    }
}
```

- *Better*

```php
final class DefaultOrder implements Order
{
    /**
     * @throws Exception
     */
    public function sell(): void
    {
        if (true) { // something wrong
            throw new Exception("something wrong - can't sell");
        }

        // sell
    }

    /**
     * @throws Exception
     */
    public function decline(): void
    {
        if (true) { // something wrong
            throw new Exception("something wrong - can't decline");
        }

        // decline
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Don't configure objects

Don't send parameters to objects to configure their behavior.

Examples:

- *Bad*

```php
final class DefaultBook implements Book
{
    public function __construct(
        private readonly LogStream $stream,
        private readonly string $title,
        private readonly bool $loggable
    ) {}

    public function sell()
    {
        // sell

        if ($this->loggable) {
            $this->stream->add('book sold');
        }
    }
}
```

- *Better*

```php
final class DefaultBook implements Book
{
    public function __construct(
        private readonly BookTitle $title
    ) {}

    public function sell()
    {
        // sell
    }
}

final class LoggableBook implements Book
{
    public function __construct(
        private readonly LogStream $stream,
        private readonly DefaultBook $book
    ) {}

    public function sell()
    {
        $this->book->sell();
        $this->stream->add('book sold');
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Don't use annotations

Don't use native or DocBlock annotations if it needs a reflection.

Examples:

- *Bad*

```php
class BlogController extends AbstractController
{
    #[Route('/blog', name: 'blog_list')]
    public function list(): Response
    {
        // ...
    }
}
```

- *Better*

```php
class BlogList
{
    public function html(): Html
    {
        //
    }
}
```

## Exceptions

### Fail first instead of fail safe

Throw exceptions as soon as possible and as many as possible.

Examples:

- *Bad*

```php
class FileHelper
{
    public function list(string $dir): array
    {
        // there is no exception if there are no files in the directory, it is "fail safe" strategy
        $files = $this->listFiles($dir);

        if ($files === null) {
            throw new FileException(); // the worst thing is to ignore it
        }

        return $this->parseFileNames($dir);
    }
}
```

- *Better*

```php
interface Files
{
    public function exists(): bool;

    public function names(): FileNames;
}

final class FilesPage implements WebPage
{
    public function __construct(
        private readonly Files $files,
        private readonly FilesTemplate $template
    ) {}

    public function html(): Html
    {
        return $this->template->renderedHtml($this->files->names());
    }
}

final class StrictFilesPage implements WebPage
{
    public function __construct(
        private readonly Files $files,
        private readonly FilesPage $page,
    ) {}

    /**
     * @throws EmptyDirectoryException
     */
    public function html(): Html
    {
        // it's even better design it like "return new ThrowableIf(...)", the lines below are coupled

        if (!$this->files->exists()) {
            throw new EmptyDirectoryException();
        }

        return $this->page->html();
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Declare exceptions

Use DocBlock to declare exceptions and keep it up to date.

Examples:

- *Bad*

```php
class Foo
{
    public function bar($filePath)
    {
        if (!file_exists($file_path)) {
            throw new FilePathException();
        }

        $this->handle();
    }
}
```

- *Better*

```php
final class PostComment implements Comment
{
    public function __construct(
        private readonly Message $message,
        private readonly File $file
    ) {}

    /**
     * @throws FileNotFoundException
     * @throws FileNotLegalException
     */
    public function withFile(File $file): Comment
    {
        // it's even better wrap all cases to other objects (decorators)

        if (!$file->exists()) {
            throw new FileNotFoundException();
        }

        if (!$file->legal()) {
            throw new FileNotLegalException();
        }

        return new static($this->message, $file);
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Don't catch exceptions unless you have to

Examples:

- *Bad*

```php
class File
{
    public function getContent(): int
    {
        try {
            return $this
                ->getFile($this->path)
                ->getContent();
        } catch (FileException $e) {
            return 0;
        }
    }
}
```

- *Better*

```php
interface File
{
    public function asString(): StringType;
}

final class FilePage implements WebPage
{
    public function __construct(
        private readonly File $file,
        private readonly FileTemplate $template
    ) {}

    public function html(): StringType
    {
        try {
            return $this->template->renderedString($this->file->asString());
        } catch (FileException $exception) {
            return $this->template->failedTemplate($exception);
        }
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Chain exceptions everywhere

Rethrow exceptions everywhere you catch them and chain a new exception with previous.

Examples:

- *Bad*

```php
class Files
{
    /**
     * @throws FilesException
     */
    public function countLegal(): int
    {
        $legalFiles = 0;

        try {
            foreach($this->files as $file) {
                if ($file->isLegal()) {
                    $legalFiles++;
                }
            }
        } catch (FileException $exception) {
            throw new FilesException("can't count legal files");
        }

        return $legalFiles;
    }
}
```

- *Better*

```php
final class LegalFiles implements Files
{
    /**
     * @throws LegalFilesException
     */
    public function sendToEmails(Emails $emails): void
    {
        try {
            // it's better to create an objective mapping: new Map($this->files)
            array_map(fn($file) => $this->mail->add($emails, $file), $this->files);
        } catch (FilesException $exception) {
            throw new LegalFilesException(
                message: "Can't send to emails",
                previous: $exception
            );
        }
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Recover only at top entry points

Examples:

- *Bad*

```php
class File
{
    public function getContent(): int
    {
        try {
            return $this
                ->getFile($this->path)
                ->getContent();
        } catch (FileException $e) {
            return 0;
        }
    }
}
```

- *Better*

```php
final class WebApplication implements Application
{
    public function run(): void
    {
        try {
            echo $this->response(
                $this->templates->templateForRoute(
                    $this->route->matched()
                )
            );
        } catch (Exception $exception) {
            echo $this->response($this->templates->failedTemplate());
        }
    }
}
```

**[Table of contents ↑](#table-of-contents)**

## Patterns and principles

### RAII

Use destructors and destroy objects when you need to.

**[Table of contents ↑](#table-of-contents)**

### Factories

Create objects of **other interfaces** only in the object constructor with a condition.

> It is better to create objects of other interfaces in secondary constructors, which is not yet available in PHP.

Examples:

- *Bad*

```php
class Handler
{
    public function handle(RequestInterface $request): ResponseInterface
    {
        $helper = new Helper();
        $response = new Response();

        if ($helper->helpNeeded()) {
            $response->setSomething($helper->something());
        }

        return $response;
    }
}
```

- *Better*

```php
final class MyRule implements Rule
{
    private Foo $foo;

    public function __construct(ArrayType $instances) // you can design it better
    {
        if (array_key_exists('foo', $instances)) {
            $this->foo = $instances['foo'];
        }

        $this->foo = new Foo();
    }

    public function matches(): bool
    {
        return $this->foo->someExpected();
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Law of Demeter

Don't use the fluent interface pattern if the object just returns another object, only use it if the object creates another object.

Examples:

- *Bad*

```php
final class DefaultFoo implements Foo
{
    public function bar(): Bar
    {
        return $this->bar;
    }
}

final class DefaultBar implements Bar
{
    public function baz(): void
    {
        $this->baz();
    }
}

$foo = new DefaultFoo();
$foo->bar()->baz();
```

- *Better*

```php
final class DefaultFoo implements Foo
{
    public function __construct(
        private readonly Bar $bar
    ) { }

    public function bar(): Bar
    {
        return $this->bar->withQuux(); // creation
    }
}

final class DefaultBar implements Bar
{
    public function withQuux(): static
    {
        return new static($this->quux);
    }

    public function baz(): void
    {
        $this->baz();
    }
}

$foo = new DefaultFoo();
$foo->bar()->baz();
```

### AOP for technical moments

It is acceptable to use Aspects in annotations if the AOP functionality is not based on reflection.

Examples:

- *Bad*

```php
class Parser
{
    public function parse(): void
    {
        $attempts = 0;

        while (true) {
            try {
                $attempts++;
                $this->service->parse();
            } catch (Exception $e) {
                if ($attempts > 2) {
                    throw $e;
                }
            }
        }
    }
}
```

- *Better*

```php
final class FooAPI implements API
{
    #[RetryOnFailure(attempts: 5)]
    public function response(): Response
    {
        return $this->adapter->resposeForRequest($this->request);
    }
}
```

- *Without AOP*

```php
final class FooAPI implements API
{
    public function response(): Response
    {
        return $this->adapter->resposeForRequest($this->request);
    }
}

final class FooAPIWithAttempts implements API
{
    public function __construct(
        private readonly API $api
    ) { }

    /**
     * @return Response
     *
     * @throws Exception
     */
    public function response(): Response
    {
        // it's even better design it like "return new ObjectiveWhile(...)" or something else, because the lines below are coupled

        $attempts = 0;

        while (true) {
            try {
                $attempts++;
                return $this->api->response();
            } catch (Exception $e) {
                if ($attempts > 2) {
                    throw $e;
                }
            }
        }
    }
}
```

**[Table of contents ↑](#table-of-contents)**

### Validation

Use decorators for validation and different results (as objects) for assertions.

Examples:

- *Bad*

```php
interface FormValidator
{
    public function validate($fileds);
}
```

- *Better*

```php
interface FooReport
{
    public function asArrayBetweenDates(DateTime $from, DateTime $to): ArrayType;
}

final class FooSQLReport implements FooReport
{
    public function __construct(
        private readonly DbConnection $connection,
        private readonly TableName $tableName
    ) { }

    public function asArrayBetweenDates(DateTime $from, DateTime $to): ArrayType
    {
        return $this->connection->sqlResultAsArray(
            sql: 'SELECT * FROM ?s ...',
            tableName: this->tableName->value(),
        );
    }
}

final class StrictFooSQLReport implements FooReport
{
    public function __construct(
        private readonly FooSqlReport $report
    ) { }

    /**
     * @param DateTime $from
     * @param DateTime $to
     * @return ArrayType
     *
     * @throws Exception
     */
    public function asArrayBetweenDates(DateTime $from, DateTime $to): ArrayType
    {
        if ($from < new DateTime('now')) {
            throw new InvalidDateException();
        }

        return $this->report->asArrayBetweenDates($from, $to);
    }
}
```

**[Table of contents ↑](#table-of-contents)**
