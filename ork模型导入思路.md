---
title: ork模型导入思路
tags: ork,obj,SceneNode
grammar_cjkRuby: true
---
--------

1.先走通大结构，那就是SceneNode与mesh的兼容关系，还有drawMethod与SceneNode的关系
2.然后再纠结mesh与buffer的逻辑
3.分析SceneNode的逻辑：
--------



### 1.分析SceneNode

> 1.SceneNode
    
    A scene node is an instance of the ork::SceneNode class
    
>> 1.1 a reference frame
>> 1.2 a bounding box 用来做视锥体剪裁
>> 1.3 一系列Flag，这些flag都是自定义的，没有预先定义:you can flag some nodes as "object", others as "light", others as "transparent", others as "overlay", etc. Flags can be referenced in loops, so that you can iterate over objects flagged as "light", for instance (see Methods).
>> 1.4 一系列values，

> 2.Module

    module是为program服务的
    A module can specify some initial values for its uniform variables, and  can also specify which output varying variable must be recorded in transform feedback mode
    在 Ork中，一个 Program 由一到多个Module链接组成。

    
> 3. Value:(略)
> 4. Method:method就是为SceneNode服务的，method与task对应。
> 5. 物体绘制的执行流程：


绘制动力从SceneManager开始，在scenemanager的draw方法中，会先执行camera的drawTask：

    void SceneManager::draw()
    {
        if (camera != NULL) {
            ptr<Method> m = camera->getMethod(cameraMethod);
            if (m != NULL) {
                ptr<Task> newTask = NULL;
                try {
                    newTask = m->getTask();
                } catch (...) {
                }
                if (newTask != NULL) {
                    scheduler->run(newTask);
                    currentTask = newTask;
                } else if (currentTask != NULL) {
                    scheduler->run(currentTask);
                }
            }
        }
        ++frameNumber;
    }

看看camera的task中有什么内容，里面有各个对象的绘制调用，包括object的draw


    <sequence>
        <setState>
             <depth enable="true" value="LESS"/>
        </setState>
        <setTransforms module="this.material" worldPos="worldCameraPos"/>
        <foreach var="l" flag="light">
            <callMethod name="$l.draw"/>
        </foreach>
        <foreach var="o" flag="object" culling="true">
            <callMethod name="$o.draw"/>
        </foreach>
        <foreach var="o" flag="overlay">
            <callMethod name="$o.draw"/>
        </foreach>
    </sequence>



object的每个draw的OjbectMethod内容如下，须按照顺序先setProgram，再调用drawmesh方法，所以此处setProgram任务很关键。

    <?xml version="1.0" ?>
    <sequence> 
        <setProgram setUniforms="true">
            <module name="camera.material"/>
            <module name="light.material"/>
            <module name="this.material"/>
        </setProgram>
        <setTransforms localToScreen="localToScreen" localToWorld="localToWorld"/>
        <drawMesh name="this.geometry"/>
    </sequence>
      我们再看看具体的DrawMeshTask是如何执行的：     DrawMeshTask：
  

     bool DrawMeshTask::Impl::run()
        {
            if (m != NULL) {
                if (Logger::DEBUG_LOGGER != NULL) {
                    Resource *r = dynamic_cast<Resource*>(m.get());
                    Logger::DEBUG_LOGGER->log("SCENEGRAPH", r == NULL ? "DrawMesk" : "DrawMesh '" + r->getName() + "'");
                }
                ptr<Program> prog = SceneManager::getCurrentProgram();
                if (m->nindices == 0) {
                    SceneManager::getCurrentFrameBuffer()->**draw(prog, *m, m->mode**, 0, m->nvertices);
                } else {
                    SceneManager::getCurrentFrameBuffer()->draw(prog, *m, m->mode, 0, m->nindices);
                }
            }
            return true;
        }

    先get到当前的program，然后  draw(prog, meshbuffers, m->mode, 0, m->nindices)。


> 6. 关于task、method和AbstractTask

QualifiedName：对 形似：this.draw的抽象


### 2. 确定obj的导入结构？

一些约定：










