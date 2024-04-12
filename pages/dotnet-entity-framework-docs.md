- [Systems Data Entity](https://learn.microsoft.com/en-us/dotnet/api/system.data.entity?view=entity-framework-6.2.0)
- Mostly Database management properties can be found here
- Explored Classes
	- Database #card
		- Database to manage actual database such as connections, creating, etc.
		- Formal Definition
		  | 
		  An instance of this class is obtained from an [DbContext](https://learn.microsoft.com/en-us/dotnet/api/system.data.entity.dbcontext?view=entity-framework-6.2.0) object and can be used to manage the actual database backing a DbContext or connection. This includes creating, deleting, and checking for the existence of a database. Note that deletion and checking for existence of a database can be performed using just a connection (i.e. without a full context) by using the static methods of this class. |
	-