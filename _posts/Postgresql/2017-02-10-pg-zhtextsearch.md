---
layout: post
title:  "全文搜索 -- 从 Elk 转到 Postgres"
date:   2017-02-10 13:30:00
categories: Postgresql
comments: true
---


自己做个APP玩，里面有个中文文章搜索的功能 :  

        a. 文章有标题，标签，引用，正文等字段
        b. 根据用户输入的一些关键词，对不同字段给予不同的匹配权重，搜索出综合匹配度最高的若干篇文章


原先是用Elasticsearch，通过 ik中文分词插件 来做的，地址是 :

        https://340starobserver.github.io/2016/10/03/Elasticsearch-IK/


然Elasticsearch的效率和稳定性不强，一直在寻找替代品.

偶然上个月参加"DbGeek南京站"的活动时，听见德哥给旁边一个DBA安利pg，发现它恰巧是我一直在寻找的宝具啊


### 一.　插件 SCWS & zhparser ###

        scws 按照官网的来装就行，zhparser按照github上的来装就行
        
        装完以后 :
        
        psql -h 127.0.0.1 -p 5432 -d deepnote -U postgres
        // 我这里的数据库名字叫 deepnote
        // 注意，使用超级用户登入，否则下面创建extension会失败
        
        create extension zhparser;
        create text search configuration chinese (parser = zhparser);
        alter text search configuration chinese add mapping for n,v,a,i,e,l with simple;
        // 创建了名为 chinese 的文本搜索配置
        
        select to_tsvector('chinese','年轻人还是要提高自己的姿势水平');
        // 测试一下分词的效果


### 二.　创建文章表 ###

        create table note (
            note_id uuid not null,
            title   varchar(128) not null,
            tags    varchar(64),
            refs    text,
            feel    text,
            primary key(note_id)
        );
        // 创建一个文章表，为了说明问题，只设了编号+标题+标签+引用+正文这些字段
        // 然后准备一些像样的数据弄进去
        
        
        alter table note add column textunite tsvector;
        update note set textunite = setweight(to_tsvector('chinese',title), 'A') ||
            setweight(to_tsvector('chinese',tags) , 'B') ||
            setweight(to_tsvector('chinese',refs) , 'D') ||
            setweight(to_tsvector('chinese',feel) , 'C');
        // 添加新的一列，并使用这个单独的一列来做文本搜索( 相比多列，单列查得更快一些 )
        // 注意，这个新列 = 标题标签正文等字段的加权综合
        // 注意，权重只有ABCD四种值，且 A > B > C > D
        
        
        create index note_textunite_idx on note using gin(textunite);
        // 创建 gin索引，来加速全文搜索


### 三.　查询测试 ###

        select ts_rank_cd(textunite, query, 32) as score, title 
            from note, to_tsquery('chinese','华夏文明|(商鞅 & 战国)') query 
            where query @@ textunite 
            order by score desc limit 3;
        // 根据输入的一些关键词的组合，查出综合匹配度最高的前三篇文章
        // 参数中的32表示最终得分 score = 原得分/(原得分+1)，即进行归一化        
        // 其结果形如 :
          score   |           title            
        ----------+----------------------------
         0.545455 | 失才亡魏
         0.285714 | 卫鞅三说秦孝公
         0.166667 | 振聋发聩的《谏逐客书》
         0.119128 | 认知中国原生文明的基本理念

        // 权值的分配有待研究
