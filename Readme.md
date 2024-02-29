    
    Для того чтобы прийти к какой-то архитектуре проекта, нужно найти части кода,  которые 
    можно взять за основу. (Как аксиома и потом на основе  ее начинать работать)
    1) Здесь сложно ответить по архитектуре , так как изначально известно, что в коде ошибки, но характер этих ошибок неизвестен, поэтому ниже буду делать утверждения и предположения, которые по моему мнению наиболее вероятны.
    Замечания по существующему коду :
    файл ReturnOperation
    2) имя файла и имя класса (TsReturnOperation) должны (хорошая практика) совпадать
    3) используются классы, которые нигде не подключены в этом файле (например ReferencesOperation, MessagesClient и др)
    4) комментарии к методу doOperation() и к классу не отражают реальность
    5) doOperation() возвращает массив, а не void
    6) такой способ получения данных ...$data = (array)$this->getRequest('data')... вероятно не правильный,  здесь можно получить лишние данные, лучше передать их как входные параметры или они должны быть ранее получены в private переменную
 
    7) переменная $result. Наверное это не лучший способ использовать для возвращения такого результата. 
    8) в методе doOperation() фактически происходит валидация данных, этого здесь не должно быть, это нужно сделать ранее до вызова этого метода (например здесь ...if (empty((int)$notificationType)) {... , ...foreach ($templateData as $key => $tempData) {
            if (empty($tempData)) {
                throw new \Exception("Template Data ({$key}) is empty!", 500);
            }
        })... , и в других местах)
    9) именв методов не отражают их логику (например Employee::getById() , туда передается не ID и др)
    10) В коде часто используются throw new \Exception() который не видно где обрабатывается
    11) вместо числовых кодов ошибок в Exception желательно использовать имена а не числа

    12) Единственная задача метода doOperation() - это отправить email и sms. Поэтому не совсем понятно, почему этот метод может возвращать и массив с результатами выполнения и могут срабатывать исключения. 
    13) Есть синтаксические ошибки : используются переменные которых нет (например $error); вызываются пользовательские функции которых нет в текущем файле (например getResellerEmailFrom(), getEmailsByPermit() ); разные параметры, не совпадающие по месту расположения в методе MessagesClient::sendMessage() при 2-х вызовах; 
    14) Часто необосновано используется приведение типа (int) - можно столкнуться с потерей данных или логическими ошибками, или здесь например if (empty((int)$resellerId)) - какой смысл использования для empty особенно учитывая что $resellerId должен быть целым положительным numeric
    15) Сложно сказать, что на самом деле нужно, но этот код $reseller = Seller::getById((int)$resellerId); вернет экземпляр класса Contractor, а не Seller, так как в Contractor getById() возвращает  new self($resellerId); возвращает self класс, если нужен класс, на котором вызван этот метод, то нужно использовать позднее статическое связывание, используя  static вместо self. Тоже справедливо и для $cr = Employee::getById((int)$data['creatorId']);
    16) Есть код, в котором потенциально могут быть ошибки (например Status::getName((int)$data['differences']['to']) - нигде не проверяется существование ['to'] или параметр (int)$data['differences']['to'] в методе MessagesClient::sendMessage())
    17)  Использование такого типа для сообщений ("0") ..0 => [ // MessageTypes::EMAIL..  наверное не оправдано
    18) не вижу смысла в проверке  emails в части кода ..if (!empty($emailFrom) && count($emails) > 0) {..  так как дальнейший foreach это сам проверит
    10) Там где используется $client ( получаем его $client = Contractor::getById((int)$data['clientId'])) нет в классе Contractor свойств     email, Seller и др, при фейковом использовании
    20) Использование глобальной функции с именем __. Обычно это перевод. Но здесь этоа функция выполняет логику, которую не должна выполнять (в нее передаются массивы) (напрмер здесь ...$differences = __('PositionStatusHasChanged', [
                    'FROM' => Status::getName).....   , а здесь ...'subject'   => __('complaintEmployeeEmailSubject', $templateData, $resellerId)... - еще хуже). 
  
    
    файл others.php
    21) имя файла и имя класса  должны (хорошая практика) совпадать. В файле куча классов  - это плохая практика
    22) комментарии к методам  и к классам не отражают реальность
    23) класс  Contractor не имеет конструктора , но создание его экземпляра происходит с передачей параметра
        ...return new self($resellerId)...
    24) для объявленных переменных желательно указать тип  ( например private int $id; и инициализировать их)  и не использовать их как public, в случае, если они используются как фейковые то нет инициализации, хотя метод getFullName() их возвращает, что приведет к ошибкам
    25) Не во всех методах/функциях указаны типы входных параметров
    26) class Status. Зачем там публичные переменные, которые не используются ?  Не стоит так создавать массив типов ($a). Нужно избавиться от числовых значений, добавив константы и их использовать в массиве,  а также дать нормальное имя вместо $a

    Теперь более подробно и дополнительные рекомендации:

    Из особенностей этого кода можно сделать следующие выводы: цель этого метода doOperation() - отправить сообщения. Если сообщения отправлены, то возвращается массив с информацией об отправке. Если не отправлено, значит на каком-то этапе возникло исключение или др. Поэтому думаю, что имеет смысл вообще убрать исключения и возвращать всегда массив с информацией. 

    1) По архитектуре: Не хватает отдельной папки и классов, каждого в своем файле, для моделей (для Seller, Contractor, Employee); не хватает отдельных папок и классов для валидации, репозиторий (думаю что метод getById() должен возвращать экземпляр модели после поиска в базе данных,  а не просто объект класса); нет отдельного класса для Request, и др классов, Все это будет создано.
    2) Будет создан файл с именем класса TsReturnOperation
    3) Будут добавлены классы через use. Для некоторых классов будут указаны фейковые каталоги, предполагая, что они там есть
    4) Будут добавленны комментарии
    5) Нужно исправить возвращаемый тип с void на array.
    6)



    1) В коде часто используются throw new \Exception(). Считаю такую обработку ошибок неправильной. Лучше всего обрабатывать ошибки валидации так , чтобы клиенту всегда возвращался понятный ответ. Кроме того, если данные будут приходить не только с формы, а по API и код будет общий, то также удобнее будет вернуть обработанный ответ с кодом ошибки/ошибок (а не http). Также здесь нигде нет try-catch. Если исключение не будет обработано с помощью конструкции try-catch, то пользователь увидит фатальную ошибку PHP, которая содержит информацию об исключении, включая сообщение об ошибке и место, где ошибка возникла. Это в любом случае неприемлимо. Также неизвестно, может  сам вызов функции doOperation() обернут в try-catch. Поэтому из-за вышеуказанных неопределенностей валидация будет сделана без throw new \Exception()
    2) Непонятно, какую роль играет массив который находится в $result  и возвращается из метода doOperation().
    Желательно чтобы методы всегда возвращали один и тот же тип данных или null и также использование исключений.
    Также есть такой код $res = NotificationManager::send($resellerId, $client->id, NotificationEvents::CHANGE_RETURN_STATUS, (int)$data['differences']['to'], $templateData, $error); Здесь есть переменная $error, но она нигде не инициализирована. Какой  у нее тип ? И т.д.
    3) отправка клиентского уведомления (в самом низу метода doOperation(). Здесь видно из условия, что первый if - это проверка по валидации входных данных. И возникает вопрос, Можно ли сделать этот if переместить в самый верх и проверить, если false, то сразу выйти из метода
    4) непонятно что за код __('complaintEmployeeEmailSubject', $templateData, $resellerId). Обычно __() означает перевод, и туда передается строка, может еще какие-нибудь int, boollean, null. Но чтобы массив передавался ? Глобальная функция перевода должна быть простой и там не должны обрабатываться массивы.
    5) Сама отправка сообщений - MessagesClient::sendMessage() - Там этот код встречается дважды и передаваемые параметры не совпадают (не только по количеству(это допустимо), но там несовместимые параметры или некоторые пропущены)
    /*
        6) Из особенностей этого кода можно сделать следующие выводы: цель этого метода doOperation() - отправить сообщения. Если сообщения отправлены, то возвращается массив с информацией об отправке. Если не отправлено, значит на каком-то этапе возникло исключение или др. Поэтому думаю, что имеет смысл вообще убрать исключения и возвращать всегда массив с информацией. 
    */
    7) Возвращаемый из doOperation() результат. Если этот результат как-то обрабатывается, а он содержит в одном из полей сообщение об ошибке( $result['notificationClientBySms']['message'] = $error;), то логично такое сделать и для других полей, а также вынести все это в отдельное DTO Кроме того, наверное это не очень хорошая идея возвращать такой массив, так как его потом нужно будет анализировать и здесь не очень понятна логика дальнейших действий и фиксирование результата (там будет куча if), например отправлен еmail-  сотрудникам, а клиентам -нет, или наоборот, или sms не прошла.
    8)  Не очень понятна логика работы NotificationManager::send() и MessagesClient::sendMessage(). Судя по названию они отвечают за отправку сообщений. Но если посмотреть на передаваемые параметры,  в эти функции (например $templateData), то увидим, что эти методы делают много другой логики, этого не должно быть. И кроме того, MessagesClient::sendMessage не возвращает результат, что желательно в таких случаях, а  также нужно правильно заполнить массив $result, который возвращает doOperation()



    1) Поэтому рассмотрю вариант, предположу что Seller - это таблица какой-то организации (условно), в этой таблице есть поле email. И у этого организации есть сотрудники, информация о них находится в другой таблице -  Employee   и у каждого сотрудника есть уровень доступа к данным в каком-то поле. Также есть таблица Contractor (подрядчиков, клиентов). У Contractor и Employee есть поле seller_id, связи: Seller->Contractor и  Seller->Employee один ко многим. Такое предположение делаю из существующего кода и ошибок архитектуры, которые наиболее вероятны (например есть код $client = Contractor::getById(...)  и др)

    2) $client = Contractor::getById... , $reseller = Seller::getById... ,  $cr = Employee::getById...
        из этой части кода можно сделать такое допущение,  что Contractor, Seller, Employee - это экземпляры моделей
    3) Также, существование такого кода $client->Seller->id !==.... подтверждает, что Contractor это модель для таблицы базы данных и Seller тоже модель и в данном случае в модели Contractor должен быть код, который присоединит Seller  (в laravel через hasOne или др  - аналог SQL JOIN) при запросе к базе данных
    4) Далее можно сделать вывод, что Seller, Employee не наследуют от Contractor. Они все трое должны наследоваться от какого-то другого класса
    5) NotificationEvents - это простой класс и назначение его не известно, кроме как из названия.
    6) MessagesClient - отвечает за отправку сообщений
    7) Думаю, что данной ситуации добавлять работу с $_REQUEST в абстракный класс не очень хорошая идея, 
    класс ReferencesOperation предназначен для работы с какой-то логикой (doOperation()) и не должен отвечать 
    за обработку HTTP-запросов или возвращать их.  
    7) в методе слишком много new Exception  Это стоило бы вынести в какой-нибудь кастомный класс. Также не всегда выгодно выкидывать исключенияm 
    8) Не очень понятно ситуация с тем, что doOperation() возвращает $result. Куда она его возвращает ?
    9) В этом методе (doOperation()) не стоит проверять поля в $data в таком виде как это реализовано. Нужно валидацию полей вынести в отдельный класс. И здесь стоит вернуться к п8, валидацию нужно провести до вызова метода doOperation() и передать в этот метод уже провалидированные данные. Поэтому, так как неизвестно что за фреймворк (нужны внедрение зависимостей) и не известно откуда будет вызван doOperation(), создам метод для валидации beforeTsOperation(), и вызову его внутри doOperation()


    10) Предположу, что для reseller есть  минимум 2 таблицы : Seller  и еще таблица с например с email-контактами (так как идет поиск  $emails = getEmailsByPermit($resellerId, 'tsGoodsReturn')) и еще вероятно есть 3-я таблица,  иначе зачем обращаться за данными к отдельным функциям, если  в которой есть уникальные записи по $resellerId (идет поиск $emailFrom = getResellerEmailFrom($resellerId)). Также не понятно, почему есть этот код $reseller = Seller::getById((int)$resellerId); . и при этом $reseller нигде не используется, кроме как проверка на существование записи в таблице, но тогда логичнее было бы не возвращать всю запись, а значительнее быстрее бы произвелась операция SELECT COUNT ...   но тогда метод не должен называться getById((int)$resellerId).
    Поэтому рассмотрю вариант, предположу что Seller - это таблица какой-то организации (условно), в этой таблице есть поле email. И у этого организации есть сотрудники, информация о них находится в другой таблице -  Employee   и у каждого сотрудника есть уровень доступа к данным в каком-то поле. Также есть таблица Contractor (подрядчиков, клиентов). У Contractor и Employee есть поле seller_id, связи: Seller->Contractor и  Seller->Employee один ко многим


throw new \Exception('Empty notificationType', 400);
throw new \Exception('Seller not found!', 400);
throw new \Exception('сlient not found!', 400);
throw new \Exception('Creator not found!', 400);
throw new \Exception('Expert not found!', 400);

$notificationType = (int)$data['notificationType'];
if (empty((int)$notificationType)) {
            throw new \Exception('Empty notificationType', 400);
        }

        $reseller = Seller::getById((int)$resellerId);
        if ($reseller === null) {
            throw new \Exception('Seller not found!', 400);
        }
$cr = Employee::getById((int)$data['creatorId']);
        if ($cr === null) {
            throw new \Exception('Creator not found!', 400);
        }

        $et = Employee::getById((int)$data['expertId']);
        if ($et === null) {
            throw new \Exception('Expert not found!', 400);
        }



        $client->Seller->id !==....
        здесь можно сделать 2 вывода: 
        1) $client - экземпляр Contractor модели 
        2)  Seller класс модели (выше есть строка $reseller = Seller::getById((int)$resellerId);) из нее можно сделать такое допущение

        T
        $client = Contractor::getById((int)$data['clientId']);
        if ($client === null || $client->type !== Contractor::TYPE_CUSTOMER || $client->Seller->id !== $resellerId) {
            throw new \Exception('сlient not found!', 400);
        }

        $cFullName = $client->getFullName();
        if (empty($client->getFullName())) {
            $cFullName = $client->name;
        }

        $cr = Employee::getById((int)$data['creatorId']);
        if ($cr === null) {
            throw new \Exception('Creator not found!', 400);
        }

        $et = Employee::getById((int)$data['expertId']);
        if ($et === null) {
            throw new \Exception('Expert not found!', 400);
        }

        $differences = '';
        if ($notificationType === self::TYPE_NEW) {
            $differences = __('NewPositionAdded', null, $resellerId);
        } elseif ($notificationType === self::TYPE_CHANGE && !empty($data['differences'])) {
            $differences = __('PositionStatusHasChanged', [
                    'FROM' => Status::getName((int)$data['differences']['from']),
                    'TO'   => Status::getName((int)$data['differences']['to']),
                ], $resellerId);
        }

        $templateData = [
            'COMPLAINT_ID'       => (int)$data['complaintId'],
            'COMPLAINT_NUMBER'   => (string)$data['complaintNumber'],
            'CREATOR_ID'         => (int)$data['creatorId'],
            'CREATOR_NAME'       => $cr->getFullName(),
            'EXPERT_ID'          => (int)$data['expertId'],
            'EXPERT_NAME'        => $et->getFullName(),
            'CLIENT_ID'          => (int)$data['clientId'],
            'CLIENT_NAME'        => $cFullName,
            'CONSUMPTION_ID'     => (int)$data['consumptionId'],
            'CONSUMPTION_NUMBER' => (string)$data['consumptionNumber'],
            'AGREEMENT_NUMBER'   => (string)$data['agreementNumber'],
            'DATE'               => (string)$data['date'],
            'DIFFERENCES'        => $differences,
        ];

        // Если хоть одна переменная для шаблона не задана, то не отправляем уведомления
        foreach ($templateData as $key => $tempData) {
            if (empty($tempData)) {
                throw new \Exception("Template Data ({$key}) is empty!", 500);
            }
        }

        $emailFrom = getResellerEmailFrom($resellerId);
        // Получаем email сотрудников из настроек
        $emails = getEmailsByPermit($resellerId, 'tsGoodsReturn');
        if (!empty($emailFrom) && count($emails) > 0) {
            foreach ($emails as $email) {
                MessagesClient::sendMessage([
                    0 => [ // MessageTypes::EMAIL
                           'emailFrom' => $emailFrom,
                           'emailTo'   => $email,
                           'subject'   => __('complaintEmployeeEmailSubject', $templateData, $resellerId),
                           'message'   => __('complaintEmployeeEmailBody', $templateData, $resellerId),
                    ],
                ], $resellerId, NotificationEvents::CHANGE_RETURN_STATUS);
                $result['notificationEmployeeByEmail'] = true;

            }
        }

        // Шлём клиентское уведомление, только если произошла смена статуса
        if ($notificationType === self::TYPE_CHANGE && !empty($data['differences']['to'])) {
            if (!empty($emailFrom) && !empty($client->email)) {
                MessagesClient::sendMessage([
                    0 => [ // MessageTypes::EMAIL
                           'emailFrom' => $emailFrom,
                           'emailTo'   => $client->email,
                           'subject'   => __('complaintClientEmailSubject', $templateData, $resellerId),
                           'message'   => __('complaintClientEmailBody', $templateData, $resellerId),
                    ],
                ], $resellerId, $client->id, NotificationEvents::CHANGE_RETURN_STATUS, (int)$data['differences']['to']);
                $result['notificationClientByEmail'] = true;
            }

            if (!empty($client->mobile)) {
                $res = NotificationManager::send($resellerId, $client->id, NotificationEvents::CHANGE_RETURN_STATUS, (int)$data['differences']['to'], $templateData, $error);
                if ($res) {
                    $result['notificationClientBySms']['isSent'] = true;
                }
                if (!empty($error)) {
                    $result['notificationClientBySms']['message'] = $error;
                }
            }
        }

        return $result;
    }
}