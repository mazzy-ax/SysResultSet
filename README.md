# SysResultSet

[project]:https://github.com/mazzy-ax/SysResultSet
[license]:https://github.com/mazzy-ax/SysResultSet/blob/master/LICENSE

[SysResultSet][project] &ndash; класс-обертка на языке X++ для стандартного класса `ResultSet` в [Microsoft Dynamics AX 2009](ax2009), [Microsoft Dynamics AX 2012](ax2012), [Axapta 4.0](ax4) и [Axapta 3.0](ax3):

* позволяет удобно работать с `ResultSet`, читать значения по именам колонок, а не только по индексу колонки
* реализует один метод `get` для чтения значений разных типов (см. метод `::getColumn()`)
* позволяет использовать статические методы в старом коде, который использует стандартный `ResultSet`

а также обходит "особенности" стандартного `ResultSet`:

* каждое поле в стандартном resultSet можно читать только один раз. При попытке повторного чтения возникает ошибка.
* поля в стандартном resultSEt можно читать только в порядке возрастания номеров. Если сначала считать поле с номером 2, а затем попытаться считать поле с номером 1, то возникает ошибка.

Пример использования в ax2009:

```java
SysResultSet resultSet;

resultSet = SysResultSet::executeQuery(strfmt(@"
    with trans as (
        select o.AMOUNTMST, o.ACCOUNTNUM, o.TRANSDATE, o.REFRECID as RecId, o.DATAAREAID from VENDTRANSOPEN as o
        where o.DATAAREAID = %1
        union
        select -(t.AMOUNTMST - t.SETTLEAMOUNTMST) as AMOUNTMST, t.ACCOUNTNUM, t.TRANSDATE, t.RECID, t.DATAAREAID from VENDTRANS as t
        where t.DATAAREAID = %2
        )
    select sum(AmountMST) as AmountMST, ACCOUNTNUM, TRANSDATE, RecId, DATAAREAID from trans
        group by DATAAREAID, ACCOUNTNUM, TRANSDATE, RecId -- индекс t.AccountDateIdx, o.AccountDateIdx
        having sum(AmountMST) <> 0
        order by DATAAREAID, ACCOUNTNUM, TRANSDATE, RecId
    ", sql.sqlLiteral(open.dataAreaId),
       sql.sqlLiteral(trans.dataAreaId)));

    while( resultSet.next() )
    {
        info(strFmt("%1, %2, %3, %4",
                resultSet.value("AmountMST"),
                resultSet.value("AccountNum"),
                resultSet.value("TransDate", Types::Date),
                resultSet.value("RecId")),
            "", SysInfoAction_TableField::newBuffer(VendTrans::find(resultSet.value("RecId"))));
        progress.incCount();
    }
```

Автор первого варианта обертки - Роман Долгополов (rdol, [db](http://axforum.info/forums/member.php?u=2836)):
в первом варианте был реализован один универсальный метод `get` при помощи `resultSetMetaData`.

Полный рефакторинг класса-обертки - [mazzy](http://axforum.info/forums/member.php?u=10):
Struct values, binds, nameByField, copyTo, getNull, getPrev,
статические методы для выполнения запроса, для получения единственного значения,
статические методы для работы со стандартным resultSet в унаследованном коде.

Внимание! В Ax4 и старше для работы базового класса `ResultSet` требуется разрешение `SqlStatementExecutePermission`
это разрешение выдается только на сервере.

Внимание! Для импорта проекта ax3 сконвертируйте xpo-файл в кодировку Wind-1251. Github хранит и распаковывает файлы в кодировке UTF-8.

Следите за тем, чтобы объекты этого класса создавались на сервере, чтобы не генерировать лишнего трафика между клиентом и сервером.

mazzy, v1.03, 27.04.2016
