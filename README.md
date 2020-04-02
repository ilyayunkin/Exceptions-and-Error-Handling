[Оригинал](https://isocpp.org/wiki/faq/exceptions#why-exceptions)

### Зачем нужны исключения?

**Что исключения дают мне?** Общий ответ: Использование исключений при обработке ошибок делает ваш код проще, чище, надежнее. 
**Что не так со старым добрым errno и if-else?** Общий ответ: Они порождают переплетение обычного кода и кода обработки ошибок. Так ваш код становится беспорядочным истановится сложно понять, все ли ошибки вы обрабатываете ("спагетти-код", "крысиное гнездо проверок").

Многие вещи просто невозможны без исключений. Представьте, что ошибка обнаружена в конструкторе - как вы о ней сообщите? Придется бросить исключение. Это основа идиомы RAII (Выделение ресурсов есть инициализация), которая сама является основой наиболее эффективных современных техник. Задача конструктора - гарантировать инвариант класса (создать среду, в которой функции-члены работают), что обычно требует выделения ресурсов (памяти, блокировок, файлов, сокетов, ...).

Как бы мы обрабатывали ошибки, возникающие в конструкторе, не будь у нас исключений? 
```
    vector<double> v(100000);   // needs to allocate memory
    ofstream os("myfile");      // needs to open a file
```
Конструкторы классов vector или ofstream (output file stream) могли бы установить объекты в некое "плохое состояние" (ifstream так и делает по-умолчанию). Тогда бы все последующие операции были неудачны, что далеко от идеала. В случае с ofstream весь ваш вывод просто исчезает, если вы забыли проверить успешность инициализации. Для большинства классов последствия еще хуже. Как минимум, нам бы пришлось написать это:
```
    vector<double> v(100000);   // needs to allocate memory
    if (v.bad()) { /* handle error */ } // vector doesn't actually have a bad(); it relies on exceptions
    ofstream os("myfile");      // needs to open a file
    if (os.bad())  { /* handle error */ }
```

Видите эти дополнительные тесты на каждый объект? Полный бардак наступает при использовании классов, содержащих в себе несколько объектов, особенно если эти объеты взаимозависимы. Подробнее читайте в параграфе 8.3, главе 14  и приложении Е книги "C++ Programming Language" или в более научной статье [Exception safety: Concepts and techniques](http://stroustrup.com/except.pdf).

So writing constructors can be tricky without exceptions, but what about plain old functions? We can either return an error code or set a non-local variable (e.g., errno). Setting a global variable doesn’t work too well unless you test it immediately (or some other function might have re-set it). Don’t even think of that technique if you might have multiple threads accessing the global variable. The trouble with return values are that choosing the error return value can require cleverness and can be impossible:

Итак, написание конструкторов может быть непростым без исключений. А что с простыми функциями, Мы могли бы вернуть код ошибки или установить нелокальную переменную (errno, например). Передача значения через глобальную переменную не работает хорошо, сли вы не проверяете ее сразу же. Даже не думайте о таком пути, если у вас больше одного потока. Возврат ошибок через возвращаемое значение может потребовать смекалки и быть вообще невозможным:
```
    double d = my_sqrt(-1);     // return -1 in case of error
    if (d == -1) { /* handle error */ }
    int x = my_negate(INT_MIN); // Duh?
```
Любой int является корректным значением для некоего входного значения функции my_negate(). В таких случаях нам потребуется возвращать пару значений и не забывать про проверки. (See Stroustrup’s Beginning programming book)

Типичные возражения против исключений:
* "Исключения накладны". Не слишком. Современные реализации С++ сокращают накладные расходы до нескольких процентов (пусть будет 3%) в сравнении с игнорированием ошибок. Возврат кодов ошибок и их проверка тоже не бесплатны. А если вы не бросаете исключений, они совсемакладны или даже бесплатны. "Обычный" код будет работать даже быстрее,так как все расходы идут на создание исключения.
* "В JSF++ Страуструп прямо запрещает исключения". JSF++ написан для жеткого реального времени и safety-critical приложений управления полетом. Если вычисления занимают продолжительное время -кто-то может погибнуть, поэтому нам приходится гарантировать премя исполнения, но на текущем уровне назвития мы не можем дать гарантий для исключений. В данном случае запрещено даже распределение свободной памяти. Вообще говоря, рекомендации JSF++ симулируют работу исключений в ожидании дня, когда инструменты будут позволять дать больше гарантий при использовании исключений.
* "Использование конструктора, вызванного операцией new, вызывает утечку памяти". Брехня! Это бабьи басни, вызванные ошибкой в одном из компиляторов, исправленной десятилетие назад.

### Как использовать исключения?
Подробнее читайте в параграфе 8.3, главе 14  и приложении Е книги "C++ Programming Language". Приложение написано не для новичков.

В С++ исключения используются для оповещения об ошибках, которые нельзя обработать в месте их возникновения, например ошибка выделения ресурсов, запрошеных в конструкторе:
```
    class VectorInSpecialMemory {
        int sz;
        int* elem;
    public:
        VectorInSpecialMemory(int s) 
            : sz(s) 
            , elem(AllocateInSpecialMemory(s))
        { 
            if (elem == nullptr)
                throw std::bad_alloc();
        }
        ...
    };
```

Do not use exceptions as simply another way to return a value from a function. Most users assume – as the language definition encourages them to – that ** exception-handling code is error-handling code **, and implementations are optimized to reflect that assumption.

Не пользуйтесь исключениями просто для возврата значения из функции. Исключения предназначены для обработки ошибок и реализации оптимизированы в этом направлении.

Ключевая техника - RAII (resource acquisition is initialization), использующая конструкторы и деструкторы чтобы упорядочить управление ресурсами:
```
  void fct(string s)
    {
        File_handle f(s,"r");   // File_handle's constructor opens the file called "s"
        // use f
    } // here File_handle's destructor closes the file  
```
Если при использовании f бросаются исключения, деструктор будет вызван и файл будет должным образом закрыт, в отличие от типового стандартного решения:
```
    void old_fct(const char* s)
    {
        FILE* f = fopen(s,"r"); // open the file named "s"
        // use f
        fclose(f);  // close the file
    }
```    
Если при использовании f бросаются исключения или просто встречается return, файл не будет закрыт. В программах на языке Си longjmp() является дополнительной угрозой.

### Как не стоит использовать исключения?
Исключения в С++ созданы для *обработки ошибок*.
*Используйте **throw** только чтобы сообщить о возникновении ошибки, когда функция не может выполнить то, что она обещает и выполнить ее постусловия.
*Используйте **catch** только когда вы знаете, что можете обработать ошибку (иногда переводом ошибки в другой тип и бросанием исключения нового типа - например, поймав bad_alloc, можно бросить no_space_for_file_buffers).
*Не используйте **throw** для оповещения об ошибках использования - используйте **assert** или иные средства, чтобы отправить процесс в отладчик, или дайте процессу упасть, чтобы получить **crash dump** для отладки.
*Не используйте  **throw** при обнаружении неожиданного нарушения инварианта в вашем компоненте - используйте **assert** или иные средства, чтобы завершить программу. Бросание исключения не излечит порчу памяти и может повлечь дальнейшую порчу пользовательских данных.

Есть иные приложения исключений, используемые в других языках, но нестандартные для С++ и не поддерживаемые в полной мере в реазизациях  С++.

В частности, не используйте исключения для описания логики. Оператор **throw** - не посто иной способ вернуть значение из функции. Это сделает вашу программу медленной и смутит большинство С++ программистов, которые используют исключения только для обработки ошибок. Так же **throw** - плохой способ выйти из цикла.

### Как try / catch / throw могут улучшить качество ПО?  
Устранением одной из причин для условного перехода.

Широко используемая альтернатива исключениям - возврат кода ошибки, который должен быть явно проверен неким условным оператором, например, if. Например, printf(), scanf() and malloc() работают так: вызывающий должен проверить возвращенное значение, чтобы убедиться, что функция успешно отработала.

Хотя возврат кодов ощибок **иногда** есть наиболее правильный способ обработки ошибок, есть побочные эффекты от ввода дополнительных условных операторов:
* Снижение качества: хорошо известно, что условные операторы примерно в 10 раз чаще содержат ошибки, чтем какие-либо иные операторы. То есть, при прочих равных условиях, избегая дополнительных  условных переходов, вы получаете более надежный код.
* Замедление выхода на рынок: так как условные операторы являются точками ветвления, влияющими на число тестовых случаев, необходимых для тестирования белого ящика, необязательные условные переходы увеличивают продолжительность тестирования. Обычно если вы не испытываете все точки ветвления, в вашем коде останутся инструкции, которые ни разу не вызывались, пока они не попадут к заказчику / потребителю. 
* Возрастание цены разработки: поиск ошибок, исправление ошибок, тестирование требуют все больше времени с усложнением логики.

Так что, в сравнениие с оповещением об ошибках через возвращаемые значения и if, использование try / catch / throw скорее помогут создать код с меньшим числом дефектов, меньшей стоимостью разработки, более скорым выходом на рынок. Конечно, если в вашей организации нет опыта использования try / catch / throw, будет полещно использовать из сначала на опытном проекте, чтобы понять, что вы знаете, что вы делаете - всегда нужно привыкнуть к орудию до его практического применения.

### Я все еще не убежден: фрагмент кода в 4 строки показывает, что что коды ошибок ничем не хуже исключений. Почему я должен использовать исключения на более крупных проектах?
Потому что исключения лучше масштабируются, чтом коды ошибок. Вот классический пример в 4 строки:
```
try {
  f();
  // ...
} catch (std::exception& e) {
  // ...code that handles the error...
}
```
Вот то же самое,  но с оповещением об ошибках через возвращаемое значение:
```
int rc = f();
if (rc == 0) {
  // ...
} else {
  // ...code that handles the error...
}
```
Люди показывают на эти крошечные примеры и говорят "Исключения не улучшают здесь ни кодирование, ни тестирование, ни стоимость поддержки. Почему я должен брать из в реальный проект?"

Причина: исключения помогают вам в риальных приложениях. Вы, вероятно, не увидите каких-либо выгод на крошечном примере.

В реальной жизнии, код который обнаруживает ошибку обычно должен передать ошибку назад в другую функцию, которая обработает проблему. Эта передачас ошибки часто требует прохода через десятки функций — f1() вызывает f2() вызывает f3(), etc. И проблема обнаруживается в f10() или f100(). Информация об ошибке должна быть передана назад, в f1(), потому что только f() обладает контекстом, чтобы понять, что нужно сделать в такой ситуации. В интерактивной программе f1() обычно близка к main event loop, но в любом случает код, который обнаруживает проблему - обычно не тот же код, который обрабатывает проблему, и информация об ошибке должна быть передана через всю цепочку вызовов.

Исключения делают передачу ошибки проще:
```
void f1()
{
  try {
    // ...
    f2();
    // ...
  } catch (some_exception& e) {
    // ...code that handles the error...
  }
}
void f2() { ...; f3(); ...; }
void f3() { ...; f4(); ...; }
void f4() { ...; f5(); ...; }
void f5() { ...; f6(); ...; }
void f6() { ...; f7(); ...; }
void f7() { ...; f8(); ...; }
void f8() { ...; f9(); ...; }
void f9() { ...; f10(); ...; }
void f10()
{
  // ...
  if ( /*...some error condition...*/ )
    throw some_exception();
  // ...
}
```

Только код, обнаруживающий ошибку, f10() и код, обрабатывающий ошибку, f1() получают некоторую рябь.
Использовании кодов ошибок порождает много кода по передаче ошибки через каждую функцию в цепочке:
```
int f1()
{
  // ...
  int rc = f2();
  if (rc == 0) {
    // ...
  } else {
    // ...code that handles the error...
  }
}
int f2()
{
  // ...
  int rc = f3();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f3()
{
  // ...
  int rc = f4();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f4()
{
  // ...
  int rc = f5();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f5()
{
  // ...
  int rc = f6();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f6()
{
  // ...
  int rc = f7();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f7()
{
  // ...
  int rc = f8();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f8()
{
  // ...
  int rc = f9();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f9()
{
  // ...
  int rc = f10();
  if (rc != 0)
    return rc;
  // ...
  return 0;
}
int f10()
{
  // ...
  if (...some error condition...)
    return some_nonzero_error_code;
  // ...
  return 0;
}
```
Решение на базе кодов ошибок размазывает логику обработки ошибок. Функции с f2() по f9() имеют избыточный явный рукописный код для передачи ошибок наверх. 

* Это зашумляет функции с f2() по f9() избыточной логикой, что является местом возникновения ошибок.
* Увеличивается объем кода.
* Замутняется основная логика функций с f2() по f9().
* Возвращаемые значения выполняют две различные задачи: функции с f2() по f10() должны работать с вариантами “функция сработала успешно и вернула xxx” and “функция упала, код ошибки yyy”. Если типы xxx и yyy разные, иногда необходимо возвращать по ссылке некоторое дополнительное значение -  “successful” и “unsuccessful”.

На примерах выше исключения не дают огромных выгод. Но если смотреть шире, разница значительна.

Заключение: одно из преимуществ исключений - это более понятный и простой способ передачи информации об ошибке пользовательски функциям, которые могут обработать ошибку. Другая выгода состоит в том, что ваш код не будет иметь лишней обработки признака “successful” / “unsuccessful”. Крошечные примеры обычно не акцентируют внимание на этом аспекте, так что они не дают представления о реальной практике использования.

### Как исключения упрощают типы возвращаемого значения и аргументов?

Когда вы возвращаете коды ошибки, вам требуется различать два типа значений: один для успешного завершения функции и возврата результата вычислений, другой для передачи кода ошибок в пользовательский код. Если функция может породить ошибку по пяти причинам, вам нужно, как минимум, 6 различныйх возвращаемых значений: SUCCESS, различные коды для 5 ошибок.

Для простоты возьмем два варианта:
* Успешно, результат xxx.
* Ошибка, код ошибки yyy.
Простой пример: создадим клас Number, который поддерживает 4 арифметические операции: сложение, вычитание, умножение, деление. Очевидно, понадобятся перегруженные операторы. Давайте их создадим:
```
class Number {
public:
  friend Number operator+ (const Number& x, const Number& y);
  friend Number operator- (const Number& x, const Number& y);
  friend Number operator* (const Number& x, const Number& y);
  friend Number operator/ (const Number& x, const Number& y);
  // ...
};
```
Класс легко использовать:
```
void f(Number x, Number y)
{
  // ...
  Number sum  = x + y;
  Number diff = x - y;
  Number prod = x * y;
  Number quot = x / y;
  // ...
}
```
Но есть проблема - обработка ошибок. Сложение может вызвать переполнение, деление может вызвать divide-by-zero or underflow и т.д. Беда. Как нам передать и "Успешно - результат xxx", и "Ошибка, код ошибки yyy"?

С исключениями все просто. Они могут быть рассмотрены как дополнительный тип возвращаемого значения, используемй только при необходимости. Мы просто объявляем исключения и бросаем их когда нужно:

```
void f(Number x, Number y)
{
  try {
    // ...
    Number sum  = x + y;
    Number diff = x - y;
    Number prod = x * y;
    Number quot = x / y;
    // ...
  }
  catch (Number::Overflow& exception) {
    // ...code that handles overflow...
  }
  catch (Number::Underflow& exception) {
    // ...code that handles underflow...
  }
  catch (Number::DivideByZero& exception) {
    // ...code that handles divide-by-zero...
  }
}
```
Если мы возвращаем код ошибок, наша жизнь усложняется. Когда не получается впихнуть и значение, и код ошибки в объект Number, вы, скорее всего, решите использовать дополнительный аргумент для возврата по ссылке одного из значений:
```
class Number {
public:
  enum ReturnCode {
    Success,
    Overflow,
    Underflow,
    DivideByZero
  };
  Number add(const Number& y, ReturnCode& rc) const;
  Number sub(const Number& y, ReturnCode& rc) const;
  Number mul(const Number& y, ReturnCode& rc) const;
  Number div(const Number& y, ReturnCode& rc) const;
  // ...
};
```
А так вам пришлось бы использовать этот код:
```
int f(Number x, Number y)
{
  // ...
  Number::ReturnCode rc;
  Number sum = x.add(y, rc);
  if (rc == Number::Overflow) {
    // ...code that handles overflow...
    return -1;
  } else if (rc == Number::Underflow) {
    // ...code that handles underflow...
    return -1;
  } else if (rc == Number::DivideByZero) {
    // ...code that handles divide-by-zero...
    return -1;
  }
  Number diff = x.sub(y, rc);
  if (rc == Number::Overflow) {
    // ...code that handles overflow...
    return -1;
  } else if (rc == Number::Underflow) {
    // ...code that handles underflow...
    return -1;
  } else if (rc == Number::DivideByZero) {
    // ...code that handles divide-by-zero...
    return -1;
  }
  Number prod = x.mul(y, rc);
  if (rc == Number::Overflow) {
    // ...code that handles overflow...
    return -1;
  } else if (rc == Number::Underflow) {
    // ...code that handles underflow...
    return -1;
  } else if (rc == Number::DivideByZero) {
    // ...code that handles divide-by-zero...
    return -1;
  }
  Number quot = x.div(y, rc);
  if (rc == Number::Overflow) {
    // ...code that handles overflow...
    return -1;
  } else if (rc == Number::Underflow) {
    // ...code that handles underflow...
    return -1;
  } else if (rc == Number::DivideByZero) {
    // ...code that handles divide-by-zero...
    return -1;
  }
  // ...
}
```
Суть сказанного в том, что обычно возврат кодов ошибки приводит к уродованию интерфейсов функций, в частности, особенно если требудется передать больше информации об ошибках. Если возможны 5 ошибочных состояний и подробности ошибки представляются различными структурами данных, у вас может получиться действительно уродливый интерфейс функции.

Весь этот шум не возникает при использовании исключений, которые могут быть рассмотрены как дополнительное возвращаемое значение - функции автоматически прирастают новыми типами возвращаемых значений.

Замечание: не предлагайте использовать возвращаемые значения и хранение информации в статических переменных с областью видимости вроде Number::lastError(). Это не потокобезопасно. Даже если сегодня у вас только один поток, вы вряд ли пожелаете навечно ограничить кого-то в использовании вашего класса в многопоточной среде. Если вы все же хотите, вам следует писать большими буквами предупреждения для программистов о том, что ваш код не потокобезопасен и, возможно, не может быть таковым без основательного переписывания.

### В каком смысле исключения отделяют "удачный путь" от "неудачного пути"?
Это другое преимущество исключений перед кодами ошибок.

"Удачный путь" - ветвь логики при беспроблемном исполнении программы.

"Неудачный путь" - ветвь логики при возникновении проблемы.

Исключения отделяют удачный путь от неудачного пути.

Вот простой пример: функция f() должна вызвать функции g(), h(), i(), j() в последовательности, описанной ниже. Если любая из функций выполнена неуспешно с ошибкой “foo” или “bar”, f() должна мгновенно обработать ошибку и успешно завершиться. Если возникнет иная ошибка, f() должна передать ошибку в клиентскую программу.
Так выглядит код при использовании исключений:
```
void f()  // Using exceptions
{
  try {
    GResult gg = g();
    HResult hh = h();
    IResult ii = i();
    JResult jj = j();
    // ...
  }
  catch (FooError& e) {
    // ...code that handles "foo" errors...
  }
  catch (BarError& e) {
    // ...code that handles "bar" errors...
  }
}
```
"Удачный" и "неудачный" пути очевидным образом разделены. "Удачный" путь описан в теле блока **try** - этот код легко читается и просто выполняется, если ошибок не возникло. "Неудачный" путь представлен телом блока **catch** и телом блоков **catch** в пользовательском коде.

Использования возвращаемых значений для передачи ошибок, приводит к тому, что алгоритм становится непросто разглядеть. Оба пути безнадежно смешаны:
```
int f()  // Using return-codes
{
  int rc;  // "rc" stands for "return code"
  GResult gg = g(rc);
  if (rc == FooError) {
    // ...code that handles "foo" errors...
  } else if (rc == BarError) {
    // ...code that handles "bar" errors...
  } else if (rc != Success) {
    return rc;
  }
  HResult hh = h(rc);
  if (rc == FooError) {
    // ...code that handles "foo" errors...
  } else if (rc == BarError) {
    // ...code that handles "bar" errors...
  } else if (rc != Success) {
    return rc;
  }
  IResult ii = i(rc);
  if (rc == FooError) {
    // ...code that handles "foo" errors...
  } else if (rc == BarError) {
    // ...code that handles "bar" errors...
  } else if (rc != Success) {
    return rc;
  }
  JResult jj = j(rc);
  if (rc == FooError) {
    // ...code that handles "foo" errors...
  } else if (rc == BarError) {
    // ...code that handles "bar" errors...
  } else if (rc != Success) {
    return rc;
  }
  // ...
  return Success;
}
```
По причине смешения основной логики и логики обработки ошибок, тяжело разглядеть, что же делает код. Код с исключениями выглядит иначе - он почти что самодокументирован — базовая функциональность очевидна.

### Как я понял из предыдущего пункта, использовать исключения просто. Верно?

Нет, это не так.

Суть не в том, что работать с исключениями проще. Суть в том, что оно того стоит - выгода превышает расходы. 

Вот некоторые накладные расходы:
* Исключения требуют дисциплины. Для понимания дисциплины вам следует прочитать этот документ и, как минимум, одну из книг по теме.
* Исключения - не панацея. Если ваша команда не дисциплинирована, у вас будут проблемы в любом случае. Плохой работник налажает с любым инструментом.
* Исключения не универсальны, нельзя их использовать во всех случаях. Это часть дисциплины: вы должны осознавать, когда использовать исключения, а когда коды ошибок.
* Люди, чье эго уязвимо, будут сваливать проблемы на исключения, впрочем, как и на любую другую "новую" технологию. В идеале ваша команда должна быть эмоционально способна к обучению и росту.

К счастью уже накоплено много знаний по теме. Существуют миллионы строк кода, потрачено много человеко-веков на применение исключений. Практика показала, что исключения при правильном их использовании полезны.

Изучайте.

### Похоже, исключения усложнят мою жизнь. Должно быть, исключения плохи. Не во мне же проблема, верно?

Возможно, что и в вас!

Механизм исключений в С++ может быть мощнам и полезным, но если у вас есть неверные представления, результат может быть плох. Это инструмент. Используйте его по назначению и он вам поможет, но не обвиняйте инструмент , если применяете его не по назначению.

Если вы недовольны результатом, например, если ваш код излишне запутан, перегружен **try** блоками, возможно, дело в заблуждениях. В данном документе представлен список заблуждений.

Только не упрощайте все. иногда "заблуждения" являются правильными решениями в конкретных ситуациях.

* **The return-codes mindset:** исключения вынуждают программиста перегружать код массой **try** блоков. Они думают о **throw** как о привелегированном возвращаемом значении, а о **try/catch** как о блоке проверки "Если функция вернула ошибу" - они пихают **try** почти во все функции, которые могут бросать исключения.
* **Java мышление:** в Java ресурсы, не связанные с памятью, освобождаются с помощью блоков try/finally. При таком подходе в C++ появляется множество ненужных **try** блоков, которые, по сравнению с RAII засоряют код и усложняют понимание логики. По сути код постоянно переключается между удачной и неудачной логикой. При использовании RAII код более оптимистичен - он описывает удачный путь, код очистки сокрыт в деструкторах объектов, владеющих ресурсами.  Такой код и тестировать проще - владеющие ресурсами объекты могут быть проверены изолированно.
* **Организация классов исключений вокруг сущности, бросающей исключения, а не причины исключения:** представим банковское приложение, которое имеет 5 подсистем, способных бросать исключение при недостатке средств у клиента. Правильный подход - бросать исключение, представляющее суть исключительной ситуации, т.е “insufficient funds exception”. Неправильным решением было бы бросание каждой подсистемой исключения, специфичного для каждой подсистемы, бросание например подсистемой Foo исключения класса FooException, подсистемой Bar исключения класса BarException и т.д. Этот неправильный подход обычно ведет к созданию избыточных блоков  **try/catch**, например, поймать FooException, перепаковать его в BarException и бросить только что созданный объект.

Organizing the exception classes around the physical thrower rather than the logical reason for the throw: For example, in a banking app, suppose any of five subsystems might throw an exception when the customer has insufficient funds. The right approach is to throw an exception representing the reason for the throw, e.g., an “insufficient funds exception”; the wrong mindset is for each subsystem to throw a subsystem-specific exception. For example, the Foo subsystem might throw objects of class FooException, the Bar subsystem might throw objects of class BarException, etc. This often leads to extra try/catch blocks, e.g., to catch a FooException, repackage it into a BarException, then throw the latter. In general, exception classes should represent the problem, not the chunk of code that noticed the problem.
Using the bits / data within an exception object to differentiate different categories of errors: Suppose the Foo subsystem in our banking app throws exceptions for bad account numbers, for attempting to liquidate an illiquid asset, and for insufficient funds. When these three logically distinct kinds of errors are represented by the same exception class, the catchers need to say if to figure out what the problem really was. If your code wants to handle only bad account numbers, you need to catch the master exception class, then use if to determine whether it is one you really want to handle, and if not, to rethrow it. In general, the preferred approach is for the error condition’s logical category to get encoded into the type of the exception object, not into the data of the exception object.
Designing exception classes on a subsystem by subsystem basis: In the bad old days, the specific meaning of any given return-code was local to a given function or API. Just because one function uses the return-code of 3 to mean “success,” it was still perfectly acceptable for another function to use 3 to mean something entirely different, e.g., “failed due to out of memory.” Consistency has always been preferred, but often that didn’t happen because it didn’t need to happen. People coming with that mentality often treat C++ exception-handling the same way: they assume exception classes can be localized to a subsystem. That causes no end of grief, e.g., lots of extra try blocks to catch then throw a repackaged variant of the same exception. In large systems, exception hierarchies must be designed with a system-wide mindset. Exception classes cross subsystem boundaries — they are part of the intellectual glue that holds the architecture together.
Use of raw (as opposed to smart) pointers: This is actually just a special case of non-RAII coding, but I’m calling it out because it is so common. The result of using raw pointers is, as above, lots of extra try/catch blocks whose only purpose in life is to delete an object then re-throw the exception.
Confusing logical errors with runtime situations: For example, suppose you have a function f(Foo* p) that must never be called with nullptr. However you discover that somebody somewhere is sometimes passing nullptr anyway. There are two possibilities: either they are passing nullptr because they got bad data from an external user (for example, the user forgot to fill in a field and that ultimately resulted in a nullptr) or they just plain made a mistake in their own code. In the former case, you should throw an exception since it is a runtime situation (i.e., something you can’t detect by a careful code-review; it is not a bug). In the latter case, you should definitely fix the bug in the caller’s code. You can still add some code to write a message in the log-file if it ever happens again, and you can even throw an exception if it ever happens again, but you must not merely change the code within f(Foo* p); you must, must, MUST fix the code in the caller(s) of f(Foo* p).
There are other “wrong exception-handling mindsets,” but hopefully those will help you out. And remember: don’t take those as hard and fast rules. They are guidelines, and there are exceptions to each.
