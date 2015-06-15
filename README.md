# Phase1_IOT
Sending data to a remote server through ethernet shield

#include <SPI.h>
#include <Ethernet.h>
// Local Network Settings
byte mac[] = {0x00, 0x16, 0x3E, 0x49, 0xCF, 0x3A }; //Paste here the mac address
// GroveStreams Settings
char gsApiKey[] = "a981ad00-b167-39da-8441-4d8e477abc57"; //Paste here the api key
char gsComponentName[] = "Temperature"; //Name of the component to be used.
Figure 15
char gsDomain[] = "grovestreams.com"; //Domain name of the server where all the data is to be uploaded.
char gsComponentTemplateId[] = "temp"; //Tells Grovestreams what template to use when the feed initially arrives and a new component needs to be created.
//GroveStreams Stream IDs.
char gsStreamId1[] = "s1"; //Stream ID tell GroveStreams which component streams the values will be assigned to.
char gsStreamId2[] = "s2"; //In this case it is s1->Temp C and s2->Temp F.
const unsigned long updateFrequency = 20000UL; // Update frequency in milliseconds (20000 = 20 seconds).
const int temperaturePin = 0;
char samples[35];
char myIPAddress[20];
char myMac[20];
unsigned long lastSuccessfulUploadTime = 0;
int failedCounter = 0
// Initialize Arduino Ethernet Client
EthernetClient client; //object of EthernetClient is created as client
void setup()
{ Serial.begin(9600);
startEthernet();
}
void loop()
{
// Update sensor data to GroveStreams
if(millis() - lastSuccessfulUploadTime > updateFrequency)
{
updateGroveStreams();
}
}
void updateGroveStreams()
{ unsigned long connectAttemptTime = millis();
if (client.connect(gsDomain, 80))
{ char urlBuf[175];
sprintf(urlBuf,"PUT/api/feed?compTmplId=%s&compId=%s&compName=%s&api_key=%s%sHTTP/1.1",gsComponentTemplateId,myMac,gsComponentName, gsApiKey, getSamples()); // stores the formatted data as a string
Serial.println(urlBuf);
client.println(urlBuf);
client.print(F("Host: "));
client.println();
client.println(F("Connection: close"));
client.print(F("X-Forwarded-For: "));
client.println(myIPAddress);
client.println(F("Content-Type: application/json"));
client.println();
if (client.connected())
{
//Begin Report Response
while(!client.available())
{
delay(1);
}
while(client.available())
{
char c = client.read();
Serial.print(c);
}
//End Report Response
//Client is now disconnected; stop it to cleannup.
client.stop();
lastSuccessfulUploadTime = connectAttemptTime;
failedCounter = 0;
}
else
{
handleConnectionFailure();
}
}
else
{
handleConnectionFailure();
}
}
void handleConnectionFailure()
{
//Connection failed. Increase failed counter
failedCounter++;
Serial.print(F("Connection to GroveStreams Failed "));
Serial.print(failedCounter);
Serial.println(F(" times"));
delay(1000);
// Check if Arduino Ethernet needs to be restarted
if (failedCounter > 3 )
{
//Too many failures. Restart Ethernet.
startEthernet();
}
}
void startEthernet()
{
//Start or restart the Ethernet connection.
client.stop();
Serial.println(F("Connecting Arduino to network..."));
Serial.println();
//Wait for the connection to finish stopping
delay(2000);
//Connect to the network and obtain an IP address using DHCP
if (Ethernet.begin(mac) == 0)
{
Serial.println(F("DHCP Failed, reset your Arduino and try again"));
Serial.println();
}
else
{
Serial.println(F("Arduino connected to network using DHCP"));
//Set the mac and ip variables so that they can be used during sensor uploads later
Serial.print(F(" MAC: "));
Serial.println(getMacReadable());
Serial.print(F(" IP address: "));
Serial.println(getIpReadable(Ethernet.localIP()));
Serial.println();
}
}
char* getMacReadable()
{
//Convert the mac address to a readable string
sprintf(myMac, "%02x:%02x:%02x:%02x:%02x:%02x\0", mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
return myMac;
}
char* getIpReadable(IPAddress ipAddress)
{
//Convert the ip address to a readable string
unsigned char octet[4] = {0,0,0,0};
for (int i=0; i<4; i++)
{
octet[i] = ( ipAddress >> (i*8) ) & 0xFF;
}
sprintf(myIPAddress,"%d.%d.%d.%d\0",octet[0],octet[1],octet[2],octet[3]); //Taking the ip address in 4 array storing 8 bit in each according to their significance
return myIPAddress;
}
char* getSamples()
{
//Get the temperature analog reading and convert it to a string
float voltage, degreesC, degreesF,temp;
voltage = (voltage - 0.5) *100; //used for correction of the voltage
temp =analogRead(temperaturePin);
temp=temp*0.048828125; //correction factor for getting result in desired range
degreesC = temp; //storing that value in variable
degreesF = degreesC * (9.0/5.0) + 32.0; //converting degreeC to degreeF
delay(1000);
char tempC[15] = {0}; //Initialize buffer to nulls
dtostrf(degreesC, 12, 3, tempC); //Convert float to string.
char tempF[15] = {0};
dtostrf(degreesF, 12, 3, tempF); //12 indicate the min width & 3 the precision
//Assemble the samples into URL parameters which are seperated with the "&" character
sprintf(samples, "&%s=%s&%s=%s", gsStreamId1, trim(tempC), gsStreamId2, trim(tempF));
return samples; //return the value of temperature to ethernet shield
}
char* trim(char* input) //used to remove whitespace and other predefined characters from both sides of a string without changing the actual string
{
int i,j;
char *output=input;
for (i = 0, j = 0; i<strlen(input); i++,j++)
{
if (input[i]!=' ')
output[j]=input[i];
else
j--;
}
output[j]=0;
return output;
}
