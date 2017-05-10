# Security_Door_Access
#include <MFRC522.h>
#include <Servo.h>
#include <LiquidCrystal.h>
#include <SPI.h>

#define SS_PIN 10
#define RST_PIN 9
MFRC522 mfrc522(SS_PIN, RST_PIN);
MFRC522::MIFARE_Key key;
String readString;
Servo myservo; 
LiquidCrystal lcd(7, 6, 5, 4, 3, 2);
const int ST=0;
const int tap=1;
const int dem=2;
const int Error=3;
//const int servo=4;
//const int E=5;
int const T=4;
unsigned long time1;
unsigned long time2;
//char name = "Haitao";
char name[16]="Haitao Lu";
//int ledPin = 8;

void setup() {
  // put your setup code here, to run once:
 Serial.begin(9600);
 myservo.attach(8);
// pinMode(ledPin, OUTPUT);
  SPI.begin();      // Initiate  SPI bus
  mfrc522.PCD_Init();   // Initiate MFRC522

  lcd.begin(16, 2);   //Defined LCD

  for (byte i = 0; i < 6; i++) 
    {
        key.keyByte[i] = 0xFF;
    }

  Serial.println(F("Scan a MIFARE Classic PICC to demonstrate read and write."));
  Serial.print(F("Using key (for A and B):"));
  dump_byte_array(key.keyByte, MFRC522::MF_KEY_SIZE);
  Serial.println();
    
  Serial.println(F("BEWARE: Data will be written to the PICC, in sector #1"));
    byte sector         = 1;
    byte blockAddr      = 4;
    byte dataBlock[]    = {
        0x01, 0x02, 0x03, 0x04, //  1,  2,   3,  4,
        0x05, 0x06, 0x07, 0x08, //  5,  6,   7,  8,
        0x08, 0x09, 0xff, 0x0b, //  9, 10, 255, 12,
        0x0c, 0x0d, 0x0e, 0x0f  // 13, 14,  15, 16
    };
    byte dataBlock0[]    = {
        0x00, 0x00, 0x00, 0x00, //  1,  2,   3,  4,
        0x00, 0x00, 0x00, 0x00, //  5,  6,   7,  8,
        0x00, 0x00, 0x00, 0x00, //  9, 10, 255, 12,
        0x00, 0x00, 0x00, 0x00  // 13, 14,  15, 16
    };
     byte dataBlock1 [16]={};
    byte trailerBlock   = 7;
    MFRC522::StatusCode status;
      byte buffer[18];
      byte size = sizeof(buffer);

      Serial.println(F("Authenticating using key A..."));
    status = (MFRC522::StatusCode) mfrc522.PCD_Authenticate(MFRC522::PICC_CMD_MF_AUTH_KEY_A, trailerBlock, &key, &(mfrc522.uid));
    if (status != MFRC522::STATUS_OK) {
        Serial.print(F("PCD_Authenticate() failed: "));
        Serial.println(mfrc522.GetStatusCodeName(status));
        return;
    }
    CharToByte(name,dataBlock1,16);
    Serial.print(F("write name into block ")); Serial.print(blockAddr);
    Serial.println(F(" ..."));
    dump_byte_array(dataBlock1, 16); Serial.println();
    status = (MFRC522::StatusCode) mfrc522.MIFARE_Write(blockAddr, dataBlock1, 16);
    if (status != MFRC522::STATUS_OK) {
        Serial.print(F("MIFARE_Write() failed: "));
        Serial.println(mfrc522.GetStatusCodeName(status));
    }
    Serial.println();
}



void loop() {
  // put your main code here, to run repeatedly:
  static int state = ST;
   switch (state)
  {
       case ST:
      {
        lcd.clear();
        lcd.print("Welcome");
        lcd.setCursor(0, 1);
        lcd.print("Tap or use phone");
        //digitalWrite(ledPin,LOW);
        state = tap;
        break;
      }

   ///////////////////////////////////////   
      case tap:
      {  
        Serial.flush();
        while (Serial.available()) {
    delay(3);  
    char c = Serial.read();
    readString += c;    
  }
  if (readString.length() >0) {
    Serial.println(readString);
    if (readString == "open")     
    {
     // Serial.println("1");
          state = dem;
    break;
    }
    readString="";
  }
 // Look for new cards
  if ( ! mfrc522.PICC_IsNewCardPresent()) 
  {
    return;
  }
  // Select one of the cards
  if ( ! mfrc522.PICC_ReadCardSerial()) 
  {
    return;
  }
  //Show UID on serial monitor
  Serial.print("UID tag :");
  String content= "";
  byte letter;
  for (byte i = 0; i < mfrc522.uid.size; i++) 
  {
     Serial.print(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " ");
     Serial.print(mfrc522.uid.uidByte[i], HEX);
     content.concat(String(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " "));
     content.concat(String(mfrc522.uid.uidByte[i], HEX));
  }
  Serial.println();
  Serial.print("Message : ");
  content.toUpperCase();
   if (content.substring(1) == "55 51 12 53") //change here the UID of the card/cards that you want to give access
  {
    //digitalWrite(ledPin, HIGH);
     
    state = dem;
    break;
  } 
   else  
  {
    //digitalWrite(ledPin, LOW);
    state = Error;
    break;
  }
}
     case dem:
     {
          lcd.clear();
          lcd.setCursor(0,0);
          lcd.print("Authorized access");
          lcd.setCursor(0,1);
          lcd.print(name);
          time1 = millis();
          myservo.write(180);
          state = T;
          break;
     }
     case Error:
     {
      lcd.clear();
      lcd.setCursor(0,0);
      lcd.print("Access Denied");
      
      time1 = millis();
      state = T;
      break;
     }


     case T:
      {       
        time2 = millis();

        if ((time2 - time1) >= 5000)
        {
        //digitalWrite(ledPin, LOW);
          state = ST;
        }
        else
          state = T;
       break;
      }
  }
}

void dump_byte_array(byte *buffer, byte bufferSize) {
    for (byte i = 0; i < bufferSize; i++) {
        Serial.print(buffer[i] < 0x10 ? " 0" : " ");
        Serial.print(buffer[i], HEX);
    }
}
void CharToByte(char* chars, byte* bytes, unsigned int count){
    for(unsigned int i = 0; i < count; i++)
        bytes[i] = (byte)chars[i];
}
