# Jme-Jfx
在Jfx中运行jme


# 用法
创建main
main.java
```
package com.myBox;

import com.jme3.util.LWJGLBufferAllocator;
import javafx.application.Application;
import org.lwjgl.system.Configuration;

public class Main {

    public static void main(String... args) {

        // some general settings for JFX for maximum compatibility

        Configuration.GLFW_CHECK_THREAD0.set(false); // need to disable to work on macos
        Configuration.MEMORY_ALLOCATOR.set("jemalloc"); // use jemalloc
        System.setProperty("prism.lcdtext", "false"); // JavaFx

        System.setProperty(LWJGLBufferAllocator.PROPERTY_CONCURRENT_BUFFER_ALLOCATOR, "true");

        Application.launch(MyJavaFxApplication.class, args);

    }

}

```

创建MyJavaFxApplication
MyJavaFxApplication.java
```
package com.myBox;

import com.jayfella.jfx.embedded.SimpleJfxApplication;
import com.jme3.app.FlyCamAppState;
import com.jme3.app.StatsAppState;
import com.jme3.audio.AudioListenerState;
import com.jme3.system.AppSettings;
import javafx.application.Application;
import javafx.scene.Scene;
import javafx.scene.layout.StackPane;
import javafx.stage.Stage;

import java.util.concurrent.atomic.AtomicReference;

public class MyJavaFxApplication extends Application {

    @Override
    public void start(Stage primaryStage) throws Exception {

        // We need to start JME on a new thread, not on the JFX thread.
        // We could do this a million ways, but let's just be as safe as possible.
        // 我们需要在一个新线程上启动 JME，而不是在 JFX 线程上。
        // 我们可以用很多方法来做这件事，但让我们尽量确保安全。
        AtomicReference<SimpleJfxApplication> jfxApp = new AtomicReference<>();

        new Thread(new ThreadGroup("LWJGL"), () -> {

            // create a new instance of our game.
            // SimpleJfxApplication myJmeGame = new MyJmeGame();
            // 创建我们游戏的新实例。
            // SimpleJfxApplication myJmeGame = new MyJmeGame();


            // or add some appstates..
            // 或者添加一些应用状态..
            SimpleJfxApplication myJmeGame = new MyJmeGame(
                    new StatsAppState(),
                    new AudioListenerState(),
                    new FlyCamAppState()
            );

            // set our appSettings here
            //在此设置应用程序设置
            AppSettings appSettings = myJmeGame.getSettings();
            appSettings.setUseJoysticks(true);
            appSettings.setGammaCorrection(true);
            appSettings.setSamples(16);


            jfxApp.set(myJmeGame);

            // we have a specific "start" method because due to LWJGL3 behavior this method will never return.
            // If we called this method in the constructor, it would never get constructed, so we have seperated
            // the blocking line of code in a method that gets called after construction.
            
            // 我们有一个特定的 "start" 方法，因为由于 LWJGL3 的行为，该方法将永远不会返回。
            // 如果我们在构造函数中调用此方法，它将永远不会被构建，因此我们将阻塞的代码行分离到在构造之后调用的方法中。
            jfxApp.get().start();

        }, "LWJGL Render").start();
            
        // wait for the engine to initialize...
        // You can show some kind of indeterminate progress bar in a splash screen while you wait if you like...|| !jfxApp.get().isInitialized()
        Thread.sleep(10000);
        while (jfxApp.get() == null ) {
            System.err.println(jfxApp.get().isInitialized());
            Thread.sleep(10);
        }

        // The application is never going to change from hereon in, so we can just reference the actual value.
        // Just remember that any calls to JME need to be enqueued from app.enqueue(Runnable) if you are not on the JME
        // thread (e.g. you're on the JavaFX thread). Any calls to JavaFx need to be done on the JavaFX thread, or via
        // Plaform.runLater(Runnable).
        
        // 从这里开始，应用程序将不再发生变化，因此我们可以直接引用实际值。
        // 请记住，如果你不在 JME 线程上（例如，你在 JavaFX 线程上），那么任何对 JME 的调用都需要通过 app.enqueue(Runnable) 进行排队。
        // 任何对 JavaFX 的调用都需要在 JavaFX 线程上完成，或通过 Plaform.runLater(Runnable) 进行操作。
        SimpleJfxApplication app = jfxApp.get();

        primaryStage.setTitle("Test JME Embedded in JavaFx");

        StackPane root = new StackPane();

        // add the ImageView that Jme renders to...
        //将 Jme 渲染的 ImageView 添加到...
        root.getChildren().add(app.getImageView());

        primaryStage.setScene(new Scene(root, 800, 600));
        primaryStage.show();

    }

}
```
创建MyJmeGame
MyJmeGame.java
```
package com.myBox;

import com.jayfella.jfx.embedded.SimpleJfxApplication;
import com.jme3.app.state.AppState;
import com.jme3.light.DirectionalLight;
import com.jme3.material.Material;
import com.jme3.material.Materials;
import com.jme3.math.ColorRGBA;
import com.jme3.math.Vector3f;
import com.jme3.scene.Geometry;
import com.jme3.scene.shape.Box;
import com.jme3.texture.Texture;

public class MyJmeGame extends SimpleJfxApplication {

    public MyJmeGame(AppState... initialStates) {
        super(initialStates);
    }

    private Geometry box;

    @Override
    public void initApp() {
        flyCam.setDragToRotate(true);
        flyCam.setMoveSpeed(6);

        DirectionalLight directionalLight = new DirectionalLight(
                new Vector3f(-1, -1, -1).normalizeLocal(),
                ColorRGBA.White.clone()
        );

        rootNode.addLight(directionalLight);

        Texture texture = assetManager.loadTexture("com/jme3/app/Monkey.png");

        box = new Geometry("Box", new Box(1,1,1));
        box.setMaterial(new Material(assetManager, Materials.PBR));
        box.getMaterial().setTexture("BaseColorMap", texture);
        box.getMaterial().setColor("BaseColor", ColorRGBA.White);
        box.getMaterial().setFloat("Roughness", 0.001f);
        box.getMaterial().setFloat("Metallic", 0.001f);

        rootNode.attachChild(box);

    }

    @Override
    public void simpleUpdate(float tpf) {
        box.rotate(tpf * .2f, tpf * .3f, tpf * .4f);
    }

}

```
