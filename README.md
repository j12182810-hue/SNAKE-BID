# SNAKE-BID
A funny snake game
=== settings.gradle.kts ===
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "SnakeGame"
include(":app")

=== build.gradle.kts ===
plugins {
    id("com.android.application") version "8.5.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.24" apply false
}

=== app/build.gradle.kts ===
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.gasquz.snake"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.gasquz.snake"
        minSdk = 21
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.13.1")
    implementation("androidx.appcompat:appcompat:1.7.0")
}

=== app/src/main/AndroidManifest.xml ===
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="Snake"
        android:theme="@android:style/Theme.Black.NoTitleBar.Fullscreen"
        android:supportsRtl="true">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:screenOrientation="portrait"
            android:configChanges="orientation|screenSize">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>

=== app/src/main/java/com/gasquz/snake/MainActivity.kt ===
package com.gasquz.snake

import android.app.Activity
import android.os.Bundle

class MainActivity : Activity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(SnakeView(this))
    }
}

=== app/src/main/java/com/gasquz/snake/SnakeView.kt ===
package com.gasquz.snake

import android.content.Context
import android.graphics.Canvas
import android.graphics.Color
import android.graphics.Paint
import android.view.MotionEvent
import android.view.SurfaceHolder
import android.view.SurfaceView
import kotlin.random.Random

class SnakeView(context: Context) : SurfaceView(context), SurfaceHolder.Callback, Runnable {

    private var thread: Thread? = null
    @Volatile private var running = false

    private val cellSize = 48
    private var cols = 0
    private var rows = 0

    private var snake = mutableListOf(Pair(5, 5))
    private var direction = Direction.RIGHT
    private var pendingDirection = Direction.RIGHT
    private var food = Pair(10, 10)
    private var score = 0
    private var gameOver = false
    private var moveIntervalMs = 150L
    private var lastMoveTime = 0L

    private var touchStartX = 0f
    private var touchStartY = 0f

    private val paintSnake = Paint().apply { color = Color.GREEN }
    private val paintFood = Paint().apply { color = Color.RED }
    private val paintBg = Paint().apply { color = Color.BLACK }
    private val paintText = Paint().apply {
        color = Color.WHITE
        textSize = 60f
        isAntiAlias = true
    }

    enum class Direction { UP, DOWN, LEFT, RIGHT }

    init {
        holder.addCallback(this)
        isFocusable = true
    }

    override fun surfaceCreated(holder: SurfaceHolder) {
        cols = width / cellSize
        rows = height / cellSize
        spawnFood()
        running = true
        thread = Thread(this)
        thread?.start()
    }

    override fun surfaceChanged(holder: SurfaceHolder, format: Int, w: Int, h: Int) {
        cols = w / cellSize
        rows = h / cellSize
    }

    override fun surfaceDestroyed(holder: SurfaceHolder) {
        running = false
        thread?.join()
    }

    override fun run() {
        while (running) {
            val now = System.currentTimeMillis()
            if (!gameOver && now - lastMoveTime >= moveIntervalMs) {
                update()
                lastMoveTime = now
            }
            val canvas = holder.lockCanvas() ?: continue
            try {
                draw(canvas)
            } finally {
                holder.unlockCanvasAndPost(canvas)
            }
            Thread.sleep(16)
        }
    }

    private fun update() {
        direction = pendingDirection
        val head = snake.first()
        var newHead = when (direction) {
            Direction.UP -> Pair(head.first, head.second - 1)
            Direction.DOWN -> Pair(head.first, head.second + 1)
            Direction.LEFT -> Pair(head.first - 1, head.second)
            Direction.RIGHT -> Pair(head.first + 1, head.second)
        }

        newHead = Pair(
            (newHead.first + cols) % cols,
            (newHead.second + rows) % rows
        )

        if (snake.contains(newHead)) {
            gameOver = true
            return
        }

        snake.add(0, newHead)

        if (newHead == food) {
            score++
            spawnFood()
            if (moveIntervalMs > 60) moveIntervalMs -= 3
        } else {
            snake.removeAt(snake.size - 1)
        }
    }

    private fun spawnFood() {
        do {
            food = Pair(Random.nextInt(cols), Random.nextInt(rows))
        } while (snake.contains(food))
    }

    private fun draw(canvas: Canvas) {
        canvas.drawRect(0f, 0f, width.toFloat(), height.toFloat(), paintBg)

        for (segment in snake) {
            canvas.drawRect(
                (segment.first * cellSize).toFloat(),
                (segment.second * cellSize).toFloat(),
                (segment.first * cellSize + cellSize).toFloat(),
                (segment.second * cellSize + cellSize).toFloat(),
                paintSnake
            )
        }

        canvas.drawRect(
            (food.first * cellSize).toFloat(),
            (food.second * cellSize).toFloat(),
            (food.first * cellSize + cellSize).toFloat(),
            (food.second * cellSize + cellSize).toFloat(),
            paintFood
        )

        canvas.drawText("Score: $score", 20f, 70f, paintText)

        if (gameOver) {
            canvas.drawText("GAME OVER - tap to restart", 40f, (height / 2).toFloat(), paintText)
        }
    }

    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                touchStartX = event.x
                touchStartY = event.y
                if (gameOver) restart()
            }
            MotionEvent.ACTION_UP -> {
                val dx = event.x - touchStartX
                val dy = event.y - touchStartY
                if (kotlin.math.abs(dx) > kotlin.math.abs(dy)) {
                    if (dx > 50 && direction != Direction.LEFT) pendingDirection = Direction.RIGHT
                    else if (dx < -50 && direction != Direction.RIGHT) pendingDirection = Direction.LEFT
                } else {
                    if (dy > 50 && direction != Direction.UP) pendingDirection = Direction.DOWN
                    else if (dy < -50 && direction != Direction.DOWN) pendingDirection = Direction.UP
                }
            }
        }
        return true
    }

    private fun restart() {
        snake = mutableListOf(Pair(5, 5))
        direction = Direction.RIGHT
        pendingDirection = Direction.RIGHT
        score = 0
        moveIntervalMs = 150L
        gameOver = false
        spawnFood()
    }
}

=== app/src/main/res/values/strings.xml ===
<resources>
    <string name="app_name">Snake</string>
</resources>

=== app/src/main/res/values/colors.xml ===
<resources>
    <color name="ic_launcher_background">#111111</color>
</resources>

=== app/src/main/res/drawable/ic_launcher_foreground.xml ===
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="108dp"
    android:height="108dp"
    android:viewportWidth="108"
    android:viewportHeight="108">
    <path
        android:fillColor="#4CAF50"
        android:pathData="M20,20h20v20h-20z" />
    <path
        android:fillColor="#4CAF50"
        android:pathData="M40,20h20v20h-20z" />
    <path
        android:fillColor="#4CAF50"
        android:pathData="M60,40h20v20h-20z" />
    <path
        android:fillColor="#4CAF50"
        android:pathData="M60,60h20v20h-20z" />
    <path
        android:fillColor="#F44336"
        android:pathData="M75,75h12v12h-12z" />
</vector>

=== app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml ===
<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@color/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
</adaptive-icon>

=== gradle/wrapper/gradle-wrapper.properties ===
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.7-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists

=== .github/workflows/build.yml ===
name: Build APK

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Generate gradle-wrapper.jar
        run: |
          if [ ! -f gradle/wrapper/gradle-wrapper.jar ]; then
            curl -sL -o gradle/wrapper/gradle-wrapper.jar https://raw.githubusercontent.com/gradle/gradle/v8.7.0/gradle/wrapper/gradle-wrapper.jar
          fi

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Build debug APK
        run: ./gradlew assembleDebug

      - name: Upload APK artifact
        uses: actions/upload-artifact@v4
        with:
          name: snake-game-apk
          path: app/build/outputs/apk/debug/app-debug.apk

=== gradlew ===
#!/bin/sh
APP_HOME=$( cd -P "$( dirname "$0" )" && pwd -P ) || exit
APP_BASE_NAME=${0##*/}
DEFAULT_JVM_OPTS='"-Xmx64m" "-Xms64m"'

die () {
    echo
    echo "$*"
    echo
    exit 1
} >&2

if [ -n "$JAVA_HOME" ] ; then
    if [ -x "$JAVA_HOME/jre/sh/java" ] ; then
        JAVACMD=$JAVA_HOME/jre/sh/java
    else
        JAVACMD=$JAVA_HOME/bin/java
    fi
    if [ ! -x "$JAVACMD" ] ; then
        die "ERROR: JAVA_HOME is set to an invalid directory: $JAVA_HOME"
    fi
else
    JAVACMD="java"
    which java >/dev/null 2>&1 || die "ERROR: JAVA_HOME is not set and no 'java' command could be found in your PATH."
fi

exec "$JAVACMD" $DEFAULT_JVM_OPTS $JAVA_OPTS $GRADLE_OPTS \
    "-Dorg.gradle.appname=$APP_BASE_NAME" \
    -classpath "$APP_HOME/gradle/wrapper/gradle-wrapper.jar" \
    org.gradle.wrapper.GradleWrapperMain \
    "$@"
