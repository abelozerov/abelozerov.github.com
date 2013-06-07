---
layout: post
title: "Использование Entity Framework и Membership Provider в единой транзакции"
date: 2013-06-08 01:41
comments: true
categories: 
- .NET
- TransactionScope
- EntityFramework
- Программирование
---

На проекте используются стандартные ASP.NET Providers: Membership, Roles, плюс к этому есть таблица ProfileCore, в которой хранятся некоторые данные профиля. Доступ к базе осуществляется через Entity Framework 5. Весь этот коктейль абстагирован в едином классе User:

	public class User
	{
		// from Membership provider
		public string UserName { get; set; }
		
		// from ProfileCore table
		public string FirstName { get; set; }
        public string LastName { get; set; }
        
        // from Roles provider
		public List<string> Roles { get; set; }
		...
	}

####Задача
Сделать операции с User атомарными, т.е. сохранять данные объекта User в единой транзакции
<!-- more -->

####TransactionScope
В .NET существует класс System.Transactions.TransactionScope, представляющий собой высокоуровневую транзакцию. Работает на уровне соединения с базой данных, т.е. все операции с базой, проводимые в одном TransactionScope, вне зависимости от используемых инструментов, будь про Providers, LINQ to SQL, Entity Framework, ADO.NET и их смесь, выполняются в единой транзакции, **при условии, что используется одно и то же соединение с базой данных**:

	using (TransactionScope trans = new TransactionScope())
	{
		// Some stuff
		...
		trans.Complete();
	}
	
К сожалению, Entity Framework использует свой тип Connection String, и для одновременной работы EF и Providers нужны 2 разных Connection String, даже если по сути они коннектятся к одной и той же базе. Обычно это выглядит так:

	<connectionStrings>
  		<add name="CSMembership" connectionString="…" providerName="System.Data.SqlClient" />
  		<add name="CSContext" connectionString="metadata=res://*/….csdl|res://*/….ssdl|res://*/….msl;provider=System.Data.SqlClient;provider connection string=…" providerName="System.Data.EntityClient" />
	</connectionStrings>

Поэтому, если внутри TransactionScope работать с EF объектами как обычно, `trans.Complete()` на большинстве машин выдаст вам следующие ошибки:

	The underlying provider failed on Open
	
	MSDTC on server 'SERVER_NAME' is unavailable

#### Решение

Выход из ситуации - перед открытием транзакции принудительно открыть новый Connection и сделать так, чтобы его использовали и Providers, и Entity Framework:

	MetadataWorkspace workspace = new MetadataWorkspace(
		new string[] { "res://*/" },
		new Assembly[] { Assembly.GetAssembly(typeof(CSContext)) });

	var connectionString = ConfigurationManager.ConnectionStrings["CSMembership"].ToString();

	using (TransactionScope trans = new TransactionScope())
    {
        using (SqlConnection sqlConnection = new SqlConnection(connectionString))
        using (EntityConnection entityConnection = new EntityConnection(workspace, sqlConnection))
        using (var context = new CSContext(entityConnection, false))
        {
            context.Database.Connection.ConnectionString = connectionString;
            
            // Some stuff
			...
			trans.Complete();
		}
	}

Что происходит: мы собираем для Entity Framework новый EntityConnection на основе существующего и уже открытого SqlConnection, которым пользуются Providers, и создаем контекст на его основе. Теперь внутри TransactionScope единый Connection, и все работает как надо!

Буду рад ответить на любые вопросы по теме в комментариях.