# Doctrine

PHP implementation of [Elegant Objects](https://www.amazon.com/dp/B09R21DKSC?binding=paperback).

This is a theory. All code examples are abstract.

## Object names

Object and class names must be named by what they are, not what they do.  
Avoid postfix "-er".

Bad:

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

Better:

```php
final class FooSQLReport implements FooReport
{
    public function __construct(
        private readonly DbConnection $connection,
        private readonly TableName $tableName
    ) { }

    public function asArrayBetweenDates(Date $dateFrom, Date $dateTo): ArrayType
    {
        $sql = 'SELECT * FROM ?s ...';

        $query = $this
            ->connection
            ->sqlResult($sql, $this->tableName->value());

        return $query->asArray();
    }
}
```

## Multiple constructors

Object creation should be flexible. Try to keep number of object creation ways greater then number of object methods.

Emulate a "primary constructor" after others.

Bad:

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

Better:

```php
final class DefaultFoo implements Foo
{
    private SomeType $data;

    /**
     * @throws TypeException
     */
    public function __construct(array $typedData) // you can design it better
    {
        if (array_key_exists('string', $typedData)) {
            $this->data = new SomeTypeFromString($typedData['string']);
        } else if (array_key_exists('int', $typedData)) {
            $this->data = new SomeTypeFromInt($typedData['int']);
        }

        // primary constructor below

        throw new TypeException('undefined type');
    }
}
```

## Keep constructors code-free

Constructors must encapsulate data and must not manipulate it.

Bad:

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

Better:

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

    public function __construct(StringType $inputValue)
    {
        $this->value = new StringAsInteger($inputValue);
    }
}
```

## Encapsulate as little as posible

Try to encapsulate 4 or less objects in one object;

Bad:

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

Better:

```php
final class AuthorizedUser implements User
{
    private readonly PersonalInformation $personalInformation;
    private readonly Address $address;
    private readonly Role $role;
}
```

## Identification

An object must not exists without a state, and the state must be its identity.

Bad:

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

Better:

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

## Encapsulate something

Objects must not have an empty body.

Bad:

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

Better:

```php
final class Year implements StringType
{
    public function __construct(
        private DateTime $dateTime
    ) {}

    public function value(): string
    {
        return $this->dateTime->format('Y');
    }
}
```

## Always use interfaces

Make sure that all public methods in your class is an implementation of an interface.

Bad:

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

Better:

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

## Method names

Use nouns when a method will create something and return it.  
Use verbs when a method manipulates.  
Use adjectives when a method returns a boolean value (or make those values objects and use nouns).

Never mix builders and manipulators together.

Bad:

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

Better:

```php
final class FileSavingResult implements Result
{
    public function __construct(
        private readonly bool $successful = false
    ) { }

    public function from(bool $resultOf): static
    {
        return new static($resultOf);
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

    public function save(StringType $path): void
    {
        $this->result->from(file_put_contents($path, $this->file));
    }

    public function successful(): bool
    {
        return $this->result->successful();
    }
}
```

## Don't use public constants

Don't use static properties, public constants, global variables e.t.c.

Bad:

```php
class Logger
{
    public function logWarning(string $message)
    {
        $this->loggerService->warning($message . PHP_EOL); // global constant PHP_EOL
    }
}
```

Better:

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

## Build immutable objects only

Objects should be immutable.

Bad:

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

Better:

```php
final class DefaultFoo implements Foo
{
    public function __construct(
        private readonly StringType $bar
    ) {}
}
```

## Don't use NULL

Act as if NULL did not exist.

Bad:

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

Better:

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

## Sizing

Try to stay under **250** lines and lower.

Try to create **5** or less number of public methods.

Use small interfaces, one object can implement a number of them.

Objects must be very cohesive (their methods and properties should be close to each other).

## Testing

Each class should have a unit test.  
Unit test names should shows what objects really do.

Each interface should have a fake object.  
Use fake objects instead of mock and stub objects.

Bad:

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

Better:

```php
interface PgSqlUser
{
    public function userById(IntergerType $id): User;
}

class PgSqlFakeUser implements PgSqlUser
{
    public function userById(IntergerType $id): User
    {
        return $this->user;
    }
}

final class PutCommentActionTest implements TestCase
{
    public function susscessful(): void
    {
        $pgSqlFakeUser = new PgSqlFaceUser;
        $putCommentAction = new PutCommentAction($pgSqlFakeUser, 'message');

        arrest_that($putCommentAction->succsessful());
    }

    public function failed(): void
    {
        $pgSqlFakeUser = new PgSqlFaceUser;
        $putCommentAction = new PutCommentAction($pgSqlFakeUser);

        arrest_not($putCommentAction->succsessful());
    }
}
```

## Don't use static methods

Bad:

```php
class User
{
    private User $instance;
    private function __construct() {}

    public static function getInstatnce(): User
    {
        if ($this->instance === null) {
            $this->instance = new static();
        }

        return $this->instance;
    }
}
```

Better:

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

## Don't use getters and setters

Bad:

```php
class Foo
{
    private int $number;

    public function getNumber(): int
    {
        return $number;
    }

    public function setNumber(int $number): static
    {
        $this->number = $number;

        return $this;
    }
}
```

Better:

```php
final class DefaultFoo implements Foo
{
    public function __construct(
        private IntegerType $number
    ) {}

    public function number(): IntegerType
    {
        return $this->number;
    }
}
```

## Factories

An object can create another object of its class everywehe, but another objects only in its constructor.

> It is better to create another objects in secondary constructors, which is not yet available in PHP.

Bad:

```php
class Handler
{
    public function handle(RequestInterface $request): ResponseInterface;
    {
        return new Response();
    }
}
```

Better:

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

## No reflection

Avoid any reflections.

Bad:

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

Better:

```php
interface Foo
{
    public function quuxForBar(): Quux;

    public function quuxForBaz(): Quux;
}
```

## Fail first instead of fail safe

Bad:

```php
class FileHelper
{
    public function list(string $dir): array
    {
        // there is no exception if there are no files in the directory, it is "fail safe" strategy
        $files = $this->listFiles($dir);

        if ($files === null) {
            throw new FileException; // or ignore, it doesn't matter
        }

        return $this->parseFileNames($dir);
    }
}
```

Better:

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
        private readonly FilesTemplate $templsate
    ) {}

    /**
     * @throws EmptyDirectoryException
     */
    public function html(): Html
    {
        if (!$this->files->exists()) {
            throw new EmptyDirectoryException();
        }

        return $this->template->renderedHtml($this->files->names());
    }
}
```

## Declare exceptions

Use DocBlock to declare exceptions and keep it up to date.

Bad:

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

Better:

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

## Don't catch exceptions unless you have to

Bad:

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

Better:

```php
interface File
{
    public function asString(): StringType;
}

final class FilePage implements WebPage
{
    public function __construct(
        private readonly File $file,
        private readonly FileTemplate $templsate
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

## Chain exceptions everywhere

Rethrow exceptions everywhere you catch them and chain a new exception with previous.

Bad:

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

        return $legalFiles
    }
}
```

Better:

```php
final class LegalFiles implements Files
{
    /**
     * @throws LegalFilesException
     */
    public function sendToEmails(Emails $emails): void
    {
        try {
            foreach ($this->files as $file) {
                $this->mail->add($emails, $file);
            }
        } catch (FilesException $exception) {
            throw new LegalFilesException(
                message: "Can't send to emails",
                previous: $exception
            )
        }
    }
}
```

## Recover only at top entry points

Bad:

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

Better:

```php
final class WebApplication implements Application
{
    public function run(): void
    {
        try {
            $route = $this->route->matched();
            $html = $this->templates->templateForRoute($route);

            echo $this->response($html);
        } catch (Exception $exception) {
            echo $this->response($this->templates->failedTemplate());
        }
    }
}
```

## Use AOP for technical moments

Use aspects from aspect-oriented programming for low-level programming.

Bad:

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

Better:

```php
final class FooAPI implements API
{
    /** @RetryOnFailure(attemts="5") */
    public function response(): Response
    {
        return $this->apiAdapter->resposeForRequest($this->request);
    }
}
```

## Inheritance

Make classes final or abstract. Do not overrite abstract class methods.

Extend interfaces only

Bad:

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

Better:

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

## RAII

Use destructors and destroy objects when you need to.
