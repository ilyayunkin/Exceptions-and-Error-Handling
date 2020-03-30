[Оригинал](https://isocpp.org/wiki/faq/exceptions#why-exceptions)

### Зачем нужны исключения?

**Что исключения дают мне?** Общий ответ: Использование исключений при обработке ошибок делает ваш код проще, чище, надежнее. 
**Что не так со старым добрым errno и if-else?** Общий ответ: Они порождают переплетение обычного кода и кода обработки ошибок. Так ваш код становится беспорядочным истановится сложно понять, все ли ошибки вы обрабатываете ("спагетти-код", "крысиное гнездо проверок").

МНогие вещи просто невозможны без исключений. Представьте, что ошибка обнаружена в конструкторе - как вы о ней сообщите? Придется бросить исключение. Это основа идиомы RAII (Выделение ресурсов есть инициализация), которая сама является основой наиболее эффективных современных техник. Задача конструктора - гарантировать инвариант класса (создать среду, в которой функции-члены работают), что обычно требует выделения ресурсов (памяти, блокировок, файлов, сокетов, ...).

Imagine that we did not have exceptions, how would you deal with an error detected in a constructor? Remember that constructors are often invoked to initialize/construct objects in variables:

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
C++ exceptions are designed to support error handling.

Use throw only to signal an error (which means specifically that the function couldn’t do what it advertised, and establish its postconditions).
Use catch only to specify error handling actions when you know you can handle an error (possibly by translating it to another type and rethrowing an exception of that type, such as catching a bad_alloc and rethrowing a no_space_for_file_buffers).
Do not use throw to indicate a coding error in usage of a function. Use assert or other mechanism to either send the process into a debugger or to crash the process and collect the crash dump for the developer to debug.
Do not use throw if you discover unexpected violation of an invariant of your component, use assert or other mechanism to terminate the program. Throwing an exception will not cure memory corruption and may lead to further corruption of important user data.

Исключения в С++ созданы для *обработки ошибок*.
*Используйте **throw** только чтобы сообщить о возникновении ошибки, когда функция не может выполнить то, что она обещает и выполнить ее постусловия.
*Используйте **catch** только когда вы знаете, что можете обработать ошибку (иногда переводом ошибки в другой тип и бросанием исключения нового типа - например, поймав bad_alloc, можно бросить no_space_for_file_buffers).
*Не используйте **throw** для оповещения об ошибках использования - используйте **assert** или иные средства, чтобы отправить процесс в отладчик, или дайте процессу упасть, чтобы получить **crash dump** для отладки.
*Не используйте  **throw** при обнаружении неожиданного нарушения инварианта в вашем компоненте - используйте **assert** или иные средства, чтобы завершить программу. Бросание исключения не излечит порчу памяти и может повлечь дальнейшую порчу пользовательских данных.

