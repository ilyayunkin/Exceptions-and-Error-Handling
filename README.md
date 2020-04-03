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
* **Организация классов исключений вокруг сущности, бросающей исключения, а не причины исключения:** представим банковское приложение, которое имеет 5 подсистем, способных бросать исключение при недостатке средств у клиента. Правильный подход - бросать исключение, представляющее суть исключительной ситуации, т.е “insufficient funds exception”. Неправильным решением было бы бросание каждой подсистемой исключения, специфичного для каждой подсистемы, бросание например подсистемой Foo исключения класса FooException, подсистемой Bar исключения класса BarException и т.д. Этот неправильный подход обычно ведет к созданию избыточных блоков  **try/catch**, например, поймать FooException, перепаковать его в BarException и бросить только что созданный объект. Обычно классы исключений представляют проблему, не кусок кода, обнаруживший броблему.
* **Использование битов/данных в объекте исключения для различения различных категорий ошибок** : Пусть наше банковское приложение бросает исключения при неверном номере счета, попытке ликвидировать неликвидный актив, недостатке средств. Если все эти ошибки представляются одним классом, клиентский код должен выяснять, в чем была проблема на самом деле. Если ваш код должен обработать только исключение неверного номера счета, вам нужно поймать исключение, с помощью условного оператора определить, хотите ли вы обрабатывать это исключение и если не хоитите, то бросить исключение дальше. Обычно предпочтительный продход заключается в кодировании категории ошибки в *типе* объекта исключения, а не в его *данных*.
* **Разработка классов исключений с применением идеологии подсистем**: В старые злые времена конкретное значение любого возвращаемого значения была локальна для отдельно взятой функции или API. Если одна функция использовала код 3 в значении "SUCCESS", а другвя в значении "out of memory", все было нормально. Консистентность всегда была желательна, но ее обычно не было, потому что не было необходимости. Люди, приходящие в С++ с этим опытом обычно подходят к обработке исключений в том же ключе: они предполагают, что классы исключений могут быть локализованы в подсистеме и получают бесконечные проблемы - одно и то же исключение ловится и перепаковывается в другие классы без особой на то необходимости. Иерархии исключений в больших подсистемах должны бытьспроектированы с применением системного мышления. Классы исключений пересекают границы подсистем - это одни из сущностей, связывающих архитектуру вместе.
* **Использование "сырых" указателей**: Это лишь частный случай программирования без применения RAII, выделенный лишь потому что он слишком распространен. В следствие использования "сырых" указателей появляется множество лишних **try/catch** блоков, предназначенных исключительно для удаления объектов и проброса исключения дальше.
* **Смешение логических ошибок с ситуациями времени исполнения**: например, предположим, что у вас есть функция f (Foo * p), которая никогда не должна вызываться с nullptr. Однако вы обнаруживаете, что кто-то где-то передает nullptr. Есть два варианта: либо nullptr передается по причине неверных данных от внешнего пользователя (Например, пользователь забыл заполнить поле, что привело к возникновению nullptr), либо имеется ошибка в коде. В первом случае следует бросить исключение, так как это ситуация времени исполнения (нечто, что нельзя обнаружить при анализе кода, это не баг). В последнем случае вы должны починить вызывающий код. Вы можете добавить логирование и даже бросить исключение, но прежде всего вы должны исправить вызывающий код.

Есть, конечно, и другие заблуждения. Напоминаем: не воспринимайте выше приведенные замечания как жесткие требования. Будьте благоразумны.

### У меня много try блоков. Что мне с этим делать?
Вы можете идти по пути возврата кодов ошибок даже если вы используете **try/throw/catch**. Например, вы можете располагать **try** блоки  вокруг каждого вызова.
```
void myCode()
{
  try {
    foo();
  }
  catch (FooException& e) {
    // ...
  }
  try {
    bar();
  }
  catch (BarException& e) {
    // ...
  }
  try {
    baz();
  }
  catch (BazException& e) {
    // ...
  }
}
```
Хоть здесь и использован синтакс работы с исключениями, вцелом структура очень похожа на ситуацию с кодами ошибок и стоимость разработки/тестирования/поддержки вцелом та же. Другими словами этот подход не дает вам особых преимуществ перед возвратом кодов ошибок. Вцелом это плохой подход.

Так же можно спросить себя по поводу каждого **try** блока "Зачем он здесь?" Есть несколько вариантов ответа:
* Вы можете ответить "Я обрабатываю исключение. Мой блок **catch** обрабатывает ошибки и продолжает исполнение без бросания новых исключений. Вызывающий никогда не узнает об исключительной ситуации. Мой блок **catch** не бросает каких-либо исключений и не возвращает кодов ошибок.". Тогда вы, вероятно, можете оставить **try** блок как он есть - он, возможно, впорядке.
* Вы можете ответить "Мой блок **catch** делает ..., после чего я бросаю исключение снова.". В таком случае рассмотрите преобразование блока **try** в класс, деструктор которого сделает то, что вы делаете в блоке **catch**. Например, если ваш блок **catch**  закрывает файл, затем бросает исключение, рассмотрите замену всего этого объектом класса File, деструктор которого закроет файл. Это идиома RAII.
* Выможете ответить "Я переупаковываю исключение: я ловлю XyzException, извлекаю информацию, затем бросаю PqrException.". Если такое случается, рассмотрите  создание иерархии объектов исключений, которая не требует ловить исключения, чтобы их снова бросить. Это обычно включает расширение значения XyzException. Хотя, не заходите слишком далеко.
* Возможны другие ответы. но это самые часто встречаемые.

Главное - спросите себя "зачем?". Если вы знаете причину, возможно вы сможете найти лучший способ достижения цели.
К сожалению, в  мозгу некоторых людей идея возварат кодов ошибок засела так далеко, что они не способны видеть иные альтернативы.

### Можно ли бросать исключения из конструкторов и деструкторов?
* **Из конструкторов можно:** следует бросать исключение из конструктора, когда объект невозможно должным образом инициализировать. Нет другой удовлетворительной альтернативы в данном случае.
* **Из деструкторов нельзя:** исключение может быть брошено, но оно не должно покидать пределы деструктора. Если деструктор завершается исключение, может произойти все, что угодно, потому что базовые правила стандартной библиотеки и языка будут нарушены. Не делайте этого.
Есть предостережение: исключения не могут быть использованы для некоторых проектов жесткого реального времени. Например, см. [Стандарт кодирования C ++ для JSF](http://stroustrup.com/JSF-AV-rules.pdf).

### Как обработать ошибку при исполнении конструктора?
Бросьте исключение.

Конструкторы не возвращают значений, так что вы не можете вернуть код ошибки. Так что лучший выход - бросить исключение. Если вы не можете пользоваться исключениями, то наименьше зло - установить бит, который указывает на нерабочее состояние объекта.
У идеи зомби-объектов есть подводные камни. Вам нужно добавить функцию, которая проверяет жизнеспособность объекта, затем при создании объекта данного класса, проверять его состояние. Возможно, вы решите добавить if и в функции-члены, запретив делать что-либо с неживым объектом.

На практике зомби-подход порождает уродство. Однозначно следует предпочесть исключения. Но если это невозможно, зомби-объекты могут быть меньшим злом.

Помните: если конструктор завершился исключением, память, ассоциированная с объектом очищается - память, выделенная под сам объект не утекает. Например:
```
void f()
{
  X x;             // If X::X() throws, the memory for x itself will not leak
  Y* p = new Y();  // If Y::Y() throws, the memory for *p itself will not leak
}
```
На эту тему есть, что почитать. В частности вам нужно знать, как избежать утечек памяти, если  конструктор выделяет память, вам так же нужно знать, что случается при использовании **“placement” new** вместо обычного **new**.

### Как обработать ошибку при исполнении деструктора?
Напишите в лог. Завешните процесс. НО НЕ БРОСАЙТЕ ИСКЛЮЧЕНИЕ!

И вот почему.

В C++ нельзя бросать исключение из деструктора, вызванного в процессе "раскрутки стека", вызванной другим исключением.
Если кто-то бросил Foo(), стек будет размотан между 
```
throw Foo();
```
и
```
  }
  catch (Foo e)
  {
```

