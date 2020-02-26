**1. Activa desde SQL*Plus la auditoría de los intentos de acceso fallidos al sistema. Comprueba su funcionamiento.**

	- Los parámetros de auditoria en oracle son los siguientes:

	SQL> SHOW PARAMETER AUDIT

	NAME				     TYPE	 VALUE
	------------------------------------ ----------- ------------------------------
	audit_file_dest 		     string	 /opt/oracle/admin/orcl/adump
	audit_syslog_level		     string
	audit_sys_operations		     boolean	 TRUE
	audit_trail			     string	 DB
	unified_audit_sga_queue_size	     integer	 1048576
	SQL> 

	- Para activar auditoria que compruebe los intentos fallidos al acceder al sistema:

	SQL> AUDIT CREATE SESSION WHENEVER NOT SUCCESSFUL;

	Auditoría terminada correctamente.

	SQL> 


	- Ver auditorias activas

	SQL> select * from dba_priv_audit_opts;

	USER_NAME
	--------------------------------------------------------------------------------
	PROXY_NAME
	--------------------------------------------------------------------------------
	PRIVILEGE				 SUCCESS    FAILURE
	---------------------------------------- ---------- ----------


	CREATE SESSION				 NOT SET    BY ACCESS

	- Realizamos intencionadamente unos intentos fallidos para comprobar que funciona. Para mostrar que ha funcionado ejecutamos 	lo siguiente.

	SQL> SELECT os_username,username,extended_timestamp,action_name,returncode
	FROM dba_audit_session;  2  

	OS_USERNAME
	--------------------------------------------------------------------------------
	USERNAME
	--------------------------------------------------------------------------------
	EXTENDED_TIMESTAMP
	---------------------------------------------------------------------------
	ACTION_NAME		     RETURNCODE
	---------------------------- ----------
	oracle
	HOLA
	08/02/20 23:48:32,312532 +01:00
	LOGON				   1017


	OS_USERNAME
	--------------------------------------------------------------------------------
	USERNAME
	--------------------------------------------------------------------------------
	EXTENDED_TIMESTAMP
	---------------------------------------------------------------------------
	ACTION_NAME		     RETURNCODE
	---------------------------- ----------
	oracle
	HOLA
	08/02/20 23:48:40,482140 +01:00
	LOGON				   1017


![](https://github.com/alvarocn/auditoria/blob/master/imagenes/auditoria/1.png)


	- Para desactivar la auditoría:

	NOAUDIT CREATE SESSION WHENEVER NOT SUCCESSFUL;




**2. Realiza un procedimiento en PL/SQL que te muestre los accesos fallidos junto con el motivo de los mismos, transformando el código de error almacenado en un mensaje de texto comprensible.**

	create or replace procedure falloAcceso
	is
	v_causa	varchar2(50);
	cursor  c_fallos
	is 
	select returncode, username, timestamp
	from dba_audit_session
	where returncode !=0
	and action_name='LOGON';
	begin
	
	DBMS_OUTPUT.PUT_LINE(CHR(10)||'ACCESOS FALLIDOS');
        DBMS_OUTPUT.PUT_LINE(CHR(10)||'CAUSA'||CHR(9)||'FECHA'||CHR(9)||'USUARIO');
    	DBMS_OUTPUT.PUT_LINE(CHR(9)||'*****************************');
	FOR x IN c_fallos LOOP
        	v_causa:=motivo(x.returncode);
        	DBMS_OUTPUT.PUT_LINE(CHR(10)||CHR(9)||v_causa||CHR(9)||
        		TO_CHAR(x.timestamp,'YY/MM/DD DY HH24:MI')||CHR(9)||x.username);
	end LOOP;
	end falloAcceso;
	/


	create or replace function motivo (p_fallo number)return varchar2
	is 
	v_error	varchar(50);
	begin
		if p_fallo='1017' then
	v_error:='contraseña erronea';
	end if;
	return v_error;
	if p_fallo='28000' then
	v_error:='cuenta bloqueada';
	end if;
	return v_error;
	
	end ;
	/


![](![](https://github.com/alvarocn/auditoria/blob/master/imagenes/auditoria/2.png)


**3. Activa la auditoría de las operaciones DML realizadas por SCOTT. Comprueba su funcionamiento.**

	- Para activar la auditoria DML  realizadas por scott, ejercutamos lo siguiente.

	AUDIT INSERT TABLE, UPDATE TABLE, DELETE TABLE BY SCOTT BY ACCESS;

	BY ACCESS, realiza un registro por cada acción.
	BY SESSION, realiza un registro de las acciones por sesión iniciada.

	- Nos conectamos con scott " CONN SCOTT/TIGER ".


	SQL> insert into dept VALUES(80,'logistica','el palmar');

	1 fila creada.

	SQL> UPDATE dept SET loc='Utrera' WHERE deptno=80;

	1 fila actualizada.

	SQL> DELETE FROM dept WHERE deptno=80;

	1 fila suprimida.

	SQL> COMMIT;

	Confirmación terminada.

	SQL> 

	- Para mostrar las acciones ejecutamos la siguiente consulta, como vemos activamos anteriormente ambas auditorias, tanto por accion como se sesión.

	SQL> select OS_USERNAME, username, action_name, timestamp
	   from dba_audit_object
	    where username='SCOTT';  2    3

	OS_USERNAME
	--------------------------------------------------------------------------------
	USERNAME
	--------------------------------------------------------------------------------
	ACTION_NAME		     TIMESTAM
	---------------------------- --------
	SCOTT
	INSERT			     09/02/20

	oracle
	SCOTT
	UPDATE			     09/02/20


	OS_USERNAME
	--------------------------------------------------------------------------------
	USERNAME
	--------------------------------------------------------------------------------
	ACTION_NAME		     TIMESTAM
	---------------------------- --------
	oracle
	SCOTT
	DELETE			     09/02/20
	
	oracle
	SCOTT
	INSERT			     09/02/20

	OS_USERNAME
	--------------------------------------------------------------------------------
	USERNAME
	--------------------------------------------------------------------------------
	ACTION_NAME		     TIMESTAM
	---------------------------- --------

	oracle
	SCOTT
	UPDATE			     09/02/20

	oracle
	SCOTT

	OS_USERNAME
	--------------------------------------------------------------------------------
	USERNAME
	--------------------------------------------------------------------------------
	ACTION_NAME		     TIMESTAM
	---------------------------- --------
	DELETE			     09/02/20

![](![](https://github.com/alvarocn/auditoria/blob/master/imagenes/auditoria/12)


**4.Realiza una auditoría de grano fino para almacenar información sobre la inserción de empleados del departamento 10 en la tabla emp de scott.**

	- Creamos auditoría de grano fino.

	BEGIN
	    DBMS_FGA.ADD_POLICY (
	        object_schema      =>  'SCOTT',
	        object_name        =>  'EMP',
	        policy_name        =>  'depart10',
	        audit_condition    =>  'DEPTNO = 10',
	        statement_types    =>  'INSERT' );
	END;
	/

![](![](https://github.com/alvarocn/auditoria/blob/master/imagenes/auditoria/7.png)



	- Insertamos datos.

	SQL> INSERT INTO EMP VALUES(1000, 'camargo', 'job', 2000,TO_DATE('12-JUN-1993','DD-MON-YYYY'), 4500, NULL, 10);
 
	1 row created.

	- Para ver las políticas creadas ejecutamos la siguiente consulta:


	SQL> SELECT object_schema,object_name,policy_name,policy_text
	FROM dba_audit_policies;  2  

	OBJECT_SCHEMA
	--------------------------------------------------------------------------------
	OBJECT_NAME
	--------------------------------------------------------------------------------
	POLICY_NAME
	--------------------------------------------------------------------------------
	POLICY_TEXT
	--------------------------------------------------------------------------------
	SCOTT
	EMP
	DEPART10
	DEPTNO = 10
	

	SQL> 


![](![](https://github.com/alvarocn/auditoria/blob/master/imagenes/auditoria/9.png)


	- Ejecutamos la siguiente consulta para mostrar los datos con las políticas realizadas.

	SELECT sql_text
	FROM dba_fga_audit_trail
	WHERE policy_name='DEPART10';


	SQL> SELECT sql_text
	FROM dba_fga_audit_trail
	WHERE policy_name='DEPART10';  2    3  

	SQL_TEXT
	--------------------------------------------------------------------------------
	INSERT INTO EMP VALUES(1000, 'camargo', 'job', 2000,TO_DATE('12-JUN-1993','DD-MO
	N-YYYY'), 4500, NULL, 10)


	SQL> 

	SQL> select db_user, object_name, policy_name, sql_text
	   from dba_fga_audit_trail;  2  

	DB_USER
	--------------------------------------------------------------------------------
	OBJECT_NAME
	--------------------------------------------------------------------------------
	POLICY_NAME
	--------------------------------------------------------------------------------
	SQL_TEXT
	--------------------------------------------------------------------------------
	SCOTT
	EMP
	DEPART10
	INSERT INTO EMP VALUES(1000, 'camargo', 'job', 2000,TO_DATE('12-JUN-1993','DD-MO
	N-YYYY'), 4500, NULL, 10)
	
	DB_USER
	--------------------------------------------------------------------------------
	OBJECT_NAME
	--------------------------------------------------------------------------------
	POLICY_NAME
	--------------------------------------------------------------------------------
	SQL_TEXT
	--------------------------------------------------------------------------------


	SQL> 
	

![](![](https://github.com/alvarocn/auditoria/blob/master/imagenes/auditoria/10.png)


**5.Explica la diferencia entre auditar una operación by access o by session.**

	- El comando "audit" incorpora un parámetro "by session" el registro de auditoría se escribirá una única vez por sesión y "by 	access"  realiza un registro por cada sentencia auditada.
	Para activar "by session" sería asi;

	"AUDIT INSERT TABLE, UPDATE TABLE, DELETE TABLE BY SYSTEM BY SESSION;"

	En la actividad 3 ya mostramos tanto "by access" como "by session" ya que activamos ambas, a continuación mostraremos 	solamente "by session" ya que "by access" la hemos desactivado.



	SQL> select OS_USERNAME, username, action_name, timestamp
	   from dba_audit_object
	    where username='SCOTT';  2    3  

	OS_USERNAME
	--------------------------------------------------------------------------------
	USERNAME
	--------------------------------------------------------------------------------
	ACTION_NAME		     TIMESTAM
	---------------------------- --------
	oracle
	SCOTT
	SESSION REC		     09/02/20
	
	oracle
	SCOTT
	SESSION REC		     09/02/20
	
	OS_USERNAME
	--------------------------------------------------------------------------------
	USERNAME
	--------------------------------------------------------------------------------
	ACTION_NAME		     TIMESTAM
	---------------------------- --------
	
	oracle
	SCOTT
	SESSION REC		     09/02/20

	oracle
	SCOTT
		


![](![](https://github.com/alvarocn/auditoria/blob/master/imagenes/auditoria/11.png)




**6.Documenta las diferencias entre los valores db y db, extended del parámetro audit_trail de ORACLE. Demuéstralas poniendo un ejemplo de la información sobre una operación concreta recopilada con cada uno de ellos.**


	- db: activa la auditoría y los datos se almacenarán en la taba SYS.AUD$ de Oracle.
	- db, extended: activa la auditoría y los datos se almacenarán en la taba SYS.AUD$ de Oracle. Además se escribirán los valores 	correspondientes en las columnas SQLBIND y SQLTEXT de la tabla SYS.AUD$.

	- Comprobamos su estado.

	SQL> show parameter audit;


	NAME				     TYPE	 VALUE
	------------------------------------ ----------- ------------------------------
	audit_file_dest 		     string	 /opt/oracle/admin/orcl/adump
	audit_syslog_level		     string
	audit_sys_operations		     boolean	 TRUE
	audit_trail			     string	 DB
	unified_audit_sga_queue_size	     integer	 1048576
	SQL> SQL> 
	SQL> 

	- Activamos el db extended.
	SQL> ALTER SYSTEM SET audit_trail = DB,EXTENDED SCOPE=SPFILE;

	Sistema modificado.

	SQL> 


	- Reiniciamos la instancia para ver el cambio de que db extended está activo.

	SQL> show parameter audit;

	NAME				     TYPE	 VALUE
	------------------------------------ ----------- ------------------------------
	audit_file_dest 		     string	 /opt/oracle/admin/orcl/adump
	audit_syslog_level		     string
	audit_sys_operations		     boolean	 TRUE
	audit_trail			     string	 DB, EXTENDED
	unified_audit_sga_queue_size	     integer	 1048576
	SQL> 
	

**7. Localiza en Enterprise Manager las posibilidades para realizar una auditoría e intenta repetir con dicha herramienta los apartados 1, 3 y 4.**

	En oracle 12c está limitado y no se puede realizar la tarea, más adelante lo realizaré en oracle 11.


**8. Averigua si en Postgres se pueden realizar los apartados 1, 3 y 4. Si es así, documenta el proceso adecuadamente.**

	Siguiento la documentación oficial de POSTGRES, no se pueden realizar igual que en ORACLE, la forma de poder realizarlas es a 	través de función, trigger etc en PL/PGSQL.

	Ejemplos de auditoría en POSTGRES.

	CREATE OR REPLACE FUNCTION audit.if_modified_func() RETURNS TRIGGER AS $body$
	DECLARE
	    v_old_data TEXT;
	    v_new_data TEXT;
	BEGIN

	    IF (TG_OP = 'UPDATE') THEN
	        v_old_data := ROW(OLD.*);
	        v_new_data := ROW(NEW.*);
	        INSERT INTO audit.logged_actions (schema_name,table_name,user_name,action,original_data,new_data,query) 
	        VALUES (TG_TABLE_SCHEMA::TEXT,TG_TABLE_NAME::TEXT,session_user::TEXT,substring(TG_OP,1,1),v_old_data,v_new_data, 		current_query());
	        RETURN NEW;
	    ELSIF (TG_OP = 'DELETE') THEN
	        v_old_data := ROW(OLD.*);
	        INSERT INTO audit.logged_actions (schema_name,table_name,user_name,action,original_data,query)
	        VALUES (TG_TABLE_SCHEMA::TEXT,TG_TABLE_NAME::TEXT,session_user::TEXT,substring(TG_OP,1,1),v_old_data, 		current_query());
	        RETURN OLD;
	    ELSIF (TG_OP = 'INSERT') THEN
	        v_new_data := ROW(NEW.*);				
		INSERT INTO audit.logged_actions (schema_name,table_name,user_name,action,new_data,query)
	        VALUES (TG_TABLE_SCHEMA::TEXT,TG_TABLE_NAME::TEXT,session_user::TEXT,substring(TG_OP,1,1),v_new_data, 	current_query());
        	RETURN NEW;
	    ELSE
	        RAISE WARNING '[AUDIT.IF_MODIFIED_FUNC] - Other action occurred: %, at %',TG_OP,now();
	        RETURN NULL;
	    END IF;
 
	EXCEPTION
	    WHEN data_exception THEN
	        RAISE WARNING '[AUDIT.IF_MODIFIED_FUNC] - UDF ERROR [DATA EXCEPTION] - SQLSTATE: %, SQLERRM: %',SQLSTATE,SQLERRM;
	        RETURN NULL;
	    WHEN unique_violation THEN
	        RAISE WARNING '[AUDIT.IF_MODIFIED_FUNC] - UDF ERROR [UNIQUE] - SQLSTATE: %, SQLERRM: %',SQLSTATE,SQLERRM;
	        RETURN NULL;
	    WHEN OTHERS THEN
	        RAISE WARNING '[AUDIT.IF_MODIFIED_FUNC] - UDF ERROR [OTHER] - SQLSTATE: %, SQLERRM: %',SQLSTATE,SQLERRM;
	        RETURN NULL;
	END;
	$body$
	LANGUAGE plpgsql
	SECURITY DEFINER
	SET search_path = pg_catalog, audit;

	GRANT SELECT ON audit.logged_actions TO public;
 
	CREATE INDEX logged_actions_schema_table_idx 
	ON audit.logged_actions(((schema_name||'.'||TABLE_NAME)::TEXT));
 
	CREATE INDEX logged_actions_action_tstamp_idx 
	ON audit.logged_actions(action_tstamp);
 
	CREATE INDEX logged_actions_action_idx 
	ON audit.logged_actions(action);

	CREATE schema audit;
	REVOKE CREATE ON schema audit FROM public;
 
	CREATE TABLE audit.logged_actions (
	    schema_name text NOT NULL,
	    TABLE_NAME text NOT NULL,
	    user_name text,
	    action_tstamp TIMESTAMP WITH TIME zone NOT NULL DEFAULT CURRENT_TIMESTAMP,
	    action TEXT NOT NULL CHECK (action IN ('I','D','U')),
	    original_data text,
	    new_data text,
	    query text
	) WITH (fillfactor=100);
 
	REVOKE ALL ON audit.logged_actions FROM public; 

	CREATE TRIGGER t_if_modified_trg 
	 AFTER INSERT OR UPDATE OR DELETE ON categories
	 FOR EACH ROW EXECUTE PROCEDURE audit.if_modified_func();



**9. Averigua si en MySQL se pueden realizar los apartados 1, 3 y 4. Si es así, documenta el proceso adecuadamente.**

	En MYSQL mediante trigger podemos auditar las acciones realizadas sobre una tabla.

	A continuación veremos como se hace:

	- Creamos una base de datos y creamos una tabla.

	MariaDB [(none)]> create database practica7;
	Query OK, 1 row affected (0.001 sec)

	MariaDB [(none)]> 

	MariaDB [(none)]> use practica7;
	Database changed
	MariaDB [practica7]> create table interpretes
	    -> (
	    -> id_interprete varchar(10),
	    -> nombre_interprete varchar(50),
	    -> constraint pk_interpretes primary key (id_interprete),
	    -> constraint contar check(length(id_interprete)>='5'));
	Query OK, 0 rows affected, 1 warning (0.322 sec)

	MariaDB [practica7]> 




	- Creamos la base de datos auditorias.



	MariaDB [practica7]> create database auditorias;
	Query OK, 1 row affected (0.000 sec)

	- A continuación creamos la tabla donde llegará los datos de auditorias por el trigger .

	MariaDB [auditorias]> CREATE TABLE log_accesos
	    -> (
	    -> codigo int(11) NOT NULL AUTO_INCREMENT,
	    -> usuario varchar(100),
	    -> fecha datetime,
	    -> PRIMARY KEY (`codigo`)
	    ->  )
	    -> ENGINE=MyISAM AUTO_INCREMENT=1 DEFAULT CHARSET=latin1;
	Query OK, 0 rows affected (0.029 sec)

	MariaDB [auditorias]> 

	- A continuación creamos el trigger.

	MariaDB [auditorias]> delimiter $$
	MariaDB [auditorias]> CREATE TRIGGER practica7.alvaro
	    ->   BEFORE INSERT ON practica7.interpretes
	    ->   FOR EACH ROW
	    ->   BEGIN
	    ->   INSERT INTO auditorias.log_accesos (usuario, fecha)values (CURRENT_USER(), NOW());
	    ->   END$$
	Query OK, 0 rows affected (0.062 sec)

	- Insertamos datos

	MariaDB [practica7]> insert into interpretes values ('123456','MAURICE ANDRE');
	Query OK, 1 row affected (0.013 sec)

	MariaDB [practica7]> insert into interpretes values ('1er456','CLAUDIO ARRAU');
	Query OK, 1 row affected (0.025 sec)
	
	- A continuación comprobamos que queda registrados los cambios.

	Database changed
	MariaDB [auditorias]> select * from log_accesos;
	+--------+----------------+---------------------+
	| codigo | usuario        | fecha               |
	+--------+----------------+---------------------+
	|      1 | root@localhost | 2020-02-26 20:54:42 |
	|      2 | root@localhost | 2020-02-26 20:54:50 |
	+--------+----------------+---------------------+
	2 rows in set (0.001 sec)

	MariaDB [auditorias]> 


![](![](https://github.com/alvarocn/auditoria/blob/master/imagenes/auditoria/15.png)



**10. Averigua las posibilidades que ofrece MongoDB para auditar los cambios que va sufriendo un documento.**

	- En Mongodb permite auditar tareas, para especificar cuales queremos usamos la siguiente opción.

	"-auditFilter"

	- Para auditar solo la creación de createcollection y dropcollection de colecciones.

	"{ atype: { $in: [ "createCollection", "dropCollection" ] } }"

	- Para especificar un filtro de auditoría cerramos el documento de filtro entre comillas simples para pasar el documento como 	una cadena.En formato BSON.

	"mongod –dbpath data/db –auditDestination file –auditFilter ‘{ atype: { $in: [ “createCollection”, “dropCollection” ] } }’ 	–auditFormat BSON –auditPath data/db/auditLog.bson"


**11. Averigua si en MongoDB se pueden auditar los accesos al sistema.**

	- Para auditar los accesos a la base de datos podemos utilizar:

	" { atype: "authenticate", "param.db": "test" } "

	- Para especificar un filtro de auditoría cerramos el documento de filtro entre comillas simples para pasar el documento como 	una cadena. En formato BSON.

	" mongod --dbpath data/db --auth --auditDestination file --auditFilter '{ atype: "authenticate", "param.db": "test" }' 	--auditFormat BSON --auditPath data/db/auditLog.bson "



