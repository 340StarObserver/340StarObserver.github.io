---
layout: post
title:  "mysql存储过程 & 触发器"
date:   2016-11-10 08:05:00
categories: Mysql
comments: true
---


我把做App时候的一个存储过程（邀请注册）记录下来，以便之后不用再花时间研究它的格式了  

        /*  
            procedure for regedit a new account  
            parameters  
            {  
            	usr        : the username you want to use  
            	pwd        : the password you want to use,and it is encoded to char(32) by md5  
            	registtime : the time when regedit  
            	invitor    : the username of person that he invited you to regedit a new account  
            	authcode   : the necessary code for regedit that you get from your invitor  
            	res        : 0 authcode is wrong;1 username has been used by others;2 transaction error;3 succeed  
            }  
            to make our deepnote environment good,we hope that our users are good readers,so we use invite regedit method  
            that is to say,if you want to regedit successfully,  
            A. you should get a right authcode or called invitecode  
            B. your username has not been used by others  
        */  
        
        drop procedure if exists regedit;  
        delimiter //  
        create procedure regedit (in usr varchar(16),in pwd char(32),in registtime int(11),in invitor varchar(16),in authcode char(32),out res tinyint(1))  
        begin  
            declare lastid int;  
            declare t_error int default 0;  
            declare continue handler for sqlexception set t_error=1;  
            
            start transaction;  
        	set res=0;  
        	if exists(select 1 from usertable where username=invitor and inviteauth=1 and invitecode=authcode limit 1)  
        	then  
        		if exists(select 1 from usertable where username=usr limit 1)  
        		then  
        			set res=1;  
        		else  
        			insert into usertable (username,password,birthtime) values(usr,pwd,registtime);  
        			set lastid=LAST_INSERT_ID();  
        			insert into classifytable (userid) values(lastid);  
        			set res=3;  
        		end if;  
        	end if;  
        	
        	if t_error=0 then  
                commit;  
                set res=3;  
            else  
                rollback;  
                set res=2;  
            end if;  
            
        end //  
        delimiter ;  


尝试用触发器的时候，踩过一些坑，这里记录一下  

        drop trigger if exists check_sailors;
        delimiter //
        create trigger check_sailors after insert on Sailors for each row
        begin
        	if exists(select * from Sailors 
        		where master is not NULL and master=NEW.master 
        		group by master having count(*)>2)
        	then
        		delete from Sailors where sid = NEW.sid limit 1;
        	end if;
        end //
        delimiter ;
        
        -- mysql5.6 don't support {commit, start transaction, rollback} in trigger  
        -- mysql5.6 don't support 'referencing NEW as N'  
