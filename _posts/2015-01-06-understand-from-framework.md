---
layout: post
title: Understand From Framework
categories: principle
---
###Learn From Activiti

+ 框架中有XML配置文件和配置对象（Configuration），后者通过解析前者文本得到，XML中bean的属性就对应于Configuration实现类中的field。

+ Command Pattern
  
  1.CommandExecutor,2.Command,3.CommandReceiver？（哪些是接口，哪些是实现？）
  
  命令对象，指定接收者dosomething
	
    interface Command {
	    void execute(CommandReceiver receiver);
	}
	
	interface CommandReceiver {
	    void do();
	}
	
	interface CommandExecutor {
	    void execute(Command cmd)
	}

+ Chain of Responsibility
  典型应用：拦截器 
    abstract class Handler {
      Handler next;
      abstract void execute(Request request); 
    }
	
	interface Request {
	  void do();
	}
  
  链表可否由inferface定义？
   
+ interface Query<T extends Query<?, ?>, U>

    interface ProcessDefinitionQuery 
      extends Query<ProcessDefinitionQuery, ProcessDefinition>
	
	Query的第一个泛型参数表示自身，第二个参数表示它最终查询的对象。自身对象做泛型参数，用于查询链（设置查询参数时返回自己）？
   