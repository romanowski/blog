# Intel Edison + Scala = BlinkOnboard 

For a long time programming embedded systems with limited hardware resources forced you to use native languages and run your firmware bare-metal. That effectively meant that being a Scala software house and doing embedded weren't an easy job. Fortunately, more and more relatively high-performance platforms have been recently emerging. After playing for some time with Arduino™ ecosystem, we've decided to give Intel® Edison a try - a tiny computer offered by Intel, hosting dual core Intel Atom "Tangier" for main applications, Intel Quark (not-yet-available) core for real-time use and Broadcom Wi-Fi and Bluetooth Low Energy chips. Our goal was to prove that:
-  you can actually run Scala applications on this platform,
-  Scala applications can easily be integrated with underlying hardware,
-  you are able to program with reactive paradigm in mind while exploiting low-level features like hardware interrupts.

We set out by throwing together a simple Scala application that starts blinking an on-board LED diode on pressing a hardware button and turns it off when you click it for the second time. To further validate our claims, we decided not to program asynchronous parts by hand but rather employ Akka to do it for us.


## Setting up run-time environment
To get started, we need JDK installed on our Intel® Edison.

### Installing Oracle JRE 8

```sh
export jdk=jdk-8u45-linux-i586.tar.gz
wget --continue --no-check-certificate -O ${jdk} --header "Cookie: oraclelicense=a" http://download.oracle.com/otn-pub/java/jdk/8u45-b14/${jdk}
tar -xf ${jdk}
rm ${jdk}
```

```sh
cat << EOF >> ~/.profile
export JAVA_HOME=/home/root/jdk1.8.0_45
export PATH=$PATH:${JAVA_HOME}/bin
export CLASSPATH=.:${JAVA_HOME}/jre/lib/mraa.jar
EOF
source ~/.profile
```

### Installing mraajava

Libmraa is a C/C++ low level skeleton library from Intel to interface with the IO on Edison and few other GNU/Linux embedded platforms, providing well structured API and making your high-level languages & constructs portable across supported boards. After playing around SDKs for different platforms, we really appreciate developers' intent to make communication with hardware easier and more consistent. Thanks to SWIG one can generate Java wrapper for this library, but this feature is quite new and not enabled by default.

To create required library, you can compile it on your own (takes less than one minute)...
```sh
git clone https://github.com/intel-iot-devkit/mraa.git
cd mraa
mkdir build; cd build
cmake -D BUILDSWIGNODE=OFF -D BUILDSWIGPYTHON=OFF -D BUILDSWIGJAVA=ON ..
make mraajava # takes 27 seconds
cd src/java
cp libmraajava.so /usr/lib/
cp mraa.jar ${JAVA_HOME}/jre/lib/
cd
rm -Rf mraa
```
... or download prebuild by us
```sh
wget http://edison.virtuslab.com/static/mraajava.tar.gz
tar -xf mraajava.tar.gz
cd mraajava
cp libmraajava.so /usr/lib/
cp mraa.jar ${JAVA_HOME}/jre/lib/
cd
rm -Rf mraajava{,.tar.gz}
```

## Running simple Akka-powered BlinkOnboard example

We're using Intel® Edison for Arduino Breakout Kit which has LED already connected to GPIO 13. To get started, you need to connect push button between GND, +3V and our external interrupt source - GPIO 4.
According to [Hardware Guide](www.intel.com/support/edison/sb/CS-035275.htm), describing the hardware interface of the Intel® Edison kit for Arduino expansion board, all GPIO pins are interrupt-capable.
![Connections diagram](http://virtuslab.com/wp-content/uploads/2015/05/scala_edison_1.png)

Then download our pre-build JAR and give it a try:
```sh
wget http://edison.virtuslab.com/static/BlinkOnboard-assembly-0.1.jar
java -jar BlinkOnboard-assembly-0.1.jar
```
It should start blinking after pressing the button.

[![Blinky is blinking](http://img.youtube.com/vi/bWqN4HfxY8s/0.jpg)](https://www.youtube.com/watch?v=bWqN4HfxY8s "Intel Edison + Scala = BlinkOnboard ")

#### GPIO
`Led` actor, defined in `Led.scala` is an example of simple GPIO operations - it sets adequate state on output pin using `mraa.Gpio.write`. 
```scala
class Led(ledPin: Int) extends Actor {
  import Led.Protocol._

  private val gpio = new Gpio(ledPin)
  gpio.dir(DIR_OUT)

  def ledOff: Receive = {
    case Clock  => turnOn
    case Off    =>
  }

  def ledOn: Receive = {
    case Clock  => turnOff
    case Off    => turnOff
  }

  override def receive: Receive = ledOff

  private def turnOn() = {
    write(1)
    context become ledOn
  }

  private def turnOff() = {
    write(0)
    context become ledOff
  }

  private def write(i: Int) = gpio.write(i)
}

object Led {
  def props(ledPin: Int) = Props(new Led(ledPin))

  object Protocol {
    case object Clock
    case object Off
  }
}
```

#### Setting up interrupt service routines
In `BlinkOnboard.scala` you can find an interrupt service routine for rising edge interrupt on GPIO 4 - defined as:
```scala
  private def setupButton(buttonPin: Int)(fn: => Unit) = {
    val callback = new IsrCallback {
      override def run() = fn
    }
    val gpio = new Gpio(buttonPin)
    gpio.dir(DIR_IN)
    gpio.isr(EDGE_RISING, callback, null)
  }
  val blinky = actorSystem.actorOf(Blinky.props(ledPin = 13))
  setupButton(buttonPin = 4) { blinky ! Signal }
```

`Blinky` from `Blinky.scala` is a glue between `Led` and ISR, reacting to button pressed messages and blinking the LED periodically by means of posting messages to LED actor via `akka.system.Scheduler` .
```scala
class Blinky(ledPin: Int) extends Actor {
  import Blinky.Protocol._
  import Led.Protocol._

  import context.dispatcher

  private var blinkSchedule: Option[Cancellable] = None
  private val led = context.actorOf(Led.props(ledPin))

  def notBlinking: Receive = {
    case Signal => startBlinking
  }

  def blinking: Receive = {
    case Signal => stopBlinking
  }

  override def receive: Receive = notBlinking

  private def startBlinking() = {
    blinkSchedule = Some(scheduleBlink)
    context become blinking
  }

  private def stopBlinking() = {
    blinkSchedule foreach { _.cancel() }
    blinkSchedule = None
    led ! Off
    context become notBlinking
  }

  private def scheduleBlink() = context.system.scheduler.schedule(
    initialDelay = 0.millis,
    interval = 400.millis,
    receiver = led,
    message = Clock
  )
}

object Blinky {
  def props(ledPin: Int) = Props(new Blinky(ledPin))

  object Protocol {
    case object Signal
  }
}
```

#### Tying it all together



Full project source code of this a-bit-over-engineered 1.2Hz soft-PWM is available at [VirtusLab/Edison-BlinkOnboard](https://github.com/VirtusLab/Edison-BlinkOnboard) GitHub repository.
Build using `sbt assembly` to pack all required dependencies into JAR. We're looking forward to creating more intelligent way of deploying and running Akka applications on Intel Edison, but for this simple proof-of-concept does the job just right.
