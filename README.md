# rolling-gates
arduino sketch for rolling gates
#include <RCSwitch.h>

const int pinClose = 11; // реле двигателя на закрытие
const int pinOpen = 12;  // реле двигателя на открытие
const int pinClosed = A0; // концевик полного закрытия
const int pinOpened = A1; // концевик полного открытия

const int pinRed = A2;
const int pinGreen = A3;
const int pinBlue = A4;

const int stopTimeout = 15000; // время полного закрытия/открытия
const int gateTimeout = 2000; // время для открытия калитки из полностью закрытого
const int gateCloseTimeout = gateTimeout * 1.2; // время для открытия калитки из полностью закрытого

RCSwitch mySwitch = RCSwitch();

// текущее состояние
enum t_door { 
  Closed,      // ворота полностью закрыты
  Opening,     // ворота полностью открыты
  Opened,      // ворота открываются
  Closing,     // ворота закрываются
  Gate,        // ворота в режиме калитки (узкий проход)
  StopClosing, // ворота остановили при закрывании
  StopOpening, // ворота остановили при открывании
  GateOpening, // ворота открываются в режим калитки
  Error        // ошибка, например, концевик не исправен
};

enum t_button {
  ButtonNone,
  ButtonA,
  ButtonB,
  ButtonC,
  ButtonD
};

struct t_state {
  enum t_door door;
  t_button button;
  unsigned long button_time;
  unsigned long stop_time;
} state;

void setRgb(int red, int green, int blue) {
   digitalWrite(pinRed, red ? HIGH : LOW);
   digitalWrite(pinGreen, green ? HIGH : LOW);
   digitalWrite(pinBlue, blue ? HIGH : LOW);
}

void setState(t_door neu) {
  if (state.door == neu)
    return;
    
  state.door = neu;
  switch (neu) {
    case Closed:
      Serial.println("Closed");
      setRgb(0,0,0);
      break;  
    case Opening:
      Serial.println("Opening");
      setRgb(0,1,0);
      break;
    case Opened:
      Serial.println("Opened");
      setRgb(0,0,0);
      break;
    case Closing:
      Serial.println("Closing");
      setRgb(0,0,1);
      break;
    case Gate:
      Serial.println("Gate");
      setRgb(0,0,0);
      break;
    case StopClosing:
      Serial.println("StopClosing");
      setRgb(0,0,0);
      break;
    case StopOpening:
      Serial.println("StopOpening");
      setRgb(0,0,0);
      break;
    case GateOpening:
      Serial.println("GateOpening");
      setRgb(1,1,1);
      break;
    case Error:
      Serial.println("Error");
      setRgb(1,0,0);
      break;
  }
}

void setup() {
  int closed, opened;
 
  Serial.begin(9600);
  Serial.println("Starting");
 
  mySwitch.enableReceive(0);  // Receiver on interrupt 0 => that is pin #2

  pinMode(pinClose,OUTPUT); 
  pinMode(pinOpen,OUTPUT);

  pinMode(pinClosed,INPUT); 
  pinMode(pinOpened,INPUT);

  pinMode(pinRed,OUTPUT); 
  pinMode(pinGreen,OUTPUT);
  pinMode(pinBlue,OUTPUT);

  digitalWrite(pinClose,LOW);
  digitalWrite(pinOpen,LOW);

  closed = digitalRead(pinClosed);
  opened = digitalRead(pinOpened);

  Serial.print("closed=");
  Serial.print(closed);
  Serial.print(" opened=");
  Serial.println(opened);

  if (closed && !opened) {
    setState(Closed);
  } else if (!closed && opened) {
    setState(Opened);
  } else if (!closed && !opened) {
    setState(Gate);
  } else {
    setState(Error);
  }
  Serial.println("Started");
}

void loop() {
 int closed, opened;
 unsigned long received = 0;
 unsigned long time;

 closed = digitalRead(pinClosed);
 opened = digitalRead(pinOpened);

 time = millis();
 t_button button;
 if (state.door == Opening) {
   if (opened) {
     setState(Opened);   
     digitalWrite(pinOpen, LOW);
   } else if (time > state.stop_time) {
     setState(StopClosing);   
     digitalWrite(pinOpen, LOW);     
   }
 }
 if (state.door == Closing) {
   if (closed) {
     setState(Closed);   
     digitalWrite(pinClose, LOW);
   } else if (time > state.stop_time) {
     setState(StopOpening);   
     digitalWrite(pinClose, LOW);     
   }
 }
  if (state.door == GateOpening) {
   if (time > state.stop_time) {
     setState(Gate);   
     digitalWrite(pinOpen, LOW);     
   }
 }
 if (mySwitch.available()) {
    received = mySwitch.getReceivedValue();
    mySwitch.resetAvailable();

    Serial.print("Button=");
    Serial.print(received);
    Serial.print(" at ");
    Serial.println(time);
    
    switch (received) {
      case 8571073:
        button = ButtonA;
        break;
      case 8571074: 
        button = ButtonB;
        break;
      case 8571076: 
        button = ButtonC;
        break;
      case 8571080:
        button = ButtonD;
        break;
      default:
        button = ButtonNone;
        return;
    }

    if (button == state.button && time - state.button_time < 500) {
      Serial.println("Skipped");
      return;
    }
    state.button = button;
    state.button_time = time;
    
    switch (button) {
      case ButtonA:
        if (state.door == Closed || state.door == Gate  || state.door == StopClosing) {
          setState(Opening);
          digitalWrite(pinOpen, HIGH);
          state.stop_time = time + stopTimeout;
        } else if (state.door == Opening || state.door == GateOpening) {
          setState(StopOpening);
          digitalWrite(pinOpen, LOW);           
        } else if (state.door == Closing) {
          setState(StopClosing);
          digitalWrite(pinClose, LOW);           
        } else if (state.door == Opened || state.door == StopOpening) {
          setState(Closing);
          digitalWrite(pinClose, HIGH);
          state.stop_time = time + stopTimeout;
        } 
        break;
      case ButtonB:
        if (state.door == Closed) {
          setState(GateOpening);
          digitalWrite(pinOpen, HIGH);
          state.stop_time = time + gateTimeout;          
        } else if (state.door == Gate) {
          setState(Closing);
          digitalWrite(pinClose, HIGH);
          state.stop_time = time + gateCloseTimeout;          
        }
        break;
      case ButtonC:
        break;
      case ButtonD:
        break;
      default:
        break;
    }
 }
}
