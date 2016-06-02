---
title: python学起来
tags: [develop,python]
date: 2016-06-02 14:54:44
---
# 环境搭建
win上面练习python的语法还是比较好的，但是实际执行脚本，由于很多第三方python库在win上面很难安装，建议在linux上面执行。
linux一般自带python，使用时注意对应的版本（2.x/3.x）

以下是安装的过程：
## ubuntu安装python mysql库
http://news.tuxi.com.cn/to/database/satmam/pdhsmp.html
未找到mysql配置文件解决方案：
http://www.111cn.net/sys/Ubuntu/63252.htm

# Evernote SDK接入尝试
参考官方API文档：https://dev.yinxiang.com/doc/

## tonken验证失败的问题
https://github.com/evernote/evernote-sdk-python/issues/39

## 一个第三方的操作类库
P.S. 示例中的代码片段可以参考类库中进行完善
https://github.com/littlecodersh/EasierLife/blob/master/Programs/Evernote/EvernoteController/controller.py

# Evernote接入代码
```python
#!/usr/bin/pyhton
# -*- coding: utf-8 -*-

from evernote.api.client import EvernoteClient
import evernote.edam.type.ttypes as Types
import evernote.edam.notestore.NoteStore as NoteStore

def get_user(client):
    userStore = client.get_user_store()
    user = userStore.getUser()
    return user

def create_note_book(client,bookname):
    noteStore = client.get_note_store()
    notebook = Types.Notebook()
    notebook.name = bookname
    notebook = noteStore.createNotebook(notebook)
    print notebook.guid
    
    if len(notebook.guid)>0:
        return True
    else:
        return False

def create_new_note(client,note_title,note_content,parentNotebook=None):
    noteStore = client.get_note_store()
    note = Types.Note()
    note.title = note_title
    note.content = generate_note_content(note_content)
    if parentNotebook and hasattr(parentNotebook, 'guid'):
        note.notebookGuid = parentNotebook.guid
    note = noteStore.createNote(note)  
    
def generate_note_content(content):
    result = '<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE en-note SYSTEM "http://xml.evernote.com/pub/enml2.dtd">'
    result += '<en-note>'+content+'</en-note>'
    return result
    
def get_all_books(client):
    noteStore = client.get_note_store()
    notebooks = noteStore.listNotebooks()
    return notebooks

def get_book_guid_by_name(book_list,book_name):
    for n in book_list:
        if n.name==book_name:
            return n
    return ''
    
    
#===========main===========
dev_token = "S=s21:xxxxxxx"
client = EvernoteClient(token=dev_token,sandbox=True)
client.service_host = 'app.yinxiang.com'
book_name='pythonTest'
#print get_user(client)

print '===========begin evernote task==========='
# create_result=create_note_book(client,book_name)
# if create_result==True:
#     print 'create book success'
# else:
#     print 'create book error'
    
all_book= get_all_books(client)
# for n in all_book:
#     print n.name
#     print n.guid

current_book=get_book_guid_by_name(all_book,book_name)
print 'current guid:'+current_book.guid

print 'begin create new note'
create_new_note(client,'test_title_py','my name is danny!',current_book)
print 'finish create new note'

print '===========finish task==========='
```