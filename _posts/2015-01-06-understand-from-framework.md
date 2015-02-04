---
layout: post
title: Understand From Framework
categories: principle
---
###Learn From Activiti

+ �������XML�����ļ������ö���Configuration��������ͨ������ǰ���ı��õ���XML��bean�����ԾͶ�Ӧ��Configurationʵ�����е�field��

+ Command Pattern
  
  1.CommandExecutor,2.Command,3.CommandReceiver������Щ�ǽӿڣ���Щ��ʵ�֣���
  
  �������ָ��������dosomething
	
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
  ����Ӧ�ã������� 
    abstract class Handler {
      Handler next;
      abstract void execute(Request request); 
    }
	
	interface Request {
	  void do();
	}
  
  ����ɷ���inferface���壿
   
+ interface Query<T extends Query<?, ?>, U>

    interface ProcessDefinitionQuery 
      extends Query<ProcessDefinitionQuery, ProcessDefinition>
	
	Query�ĵ�һ�����Ͳ�����ʾ�����ڶ���������ʾ�����ղ�ѯ�Ķ���������������Ͳ��������ڲ�ѯ�������ò�ѯ����ʱ�����Լ�����
   