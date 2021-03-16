'******************************************************************************
'*  AmReSi-Mathematic-Buzz-Game (Amadeus, Silas, René)
'*                     Ver.1.01
'*            Letzte Änderung 26.04.2014
'******************************************************************************


'*********************** Compilereinstellung **********************************
$regfile = "m32def.dat"                                     '### Chipsatzfile Auswahl
$framesize = 100
$swstack = 100
$hwstack = 100
$crystal = 16000000                                         '### Taktfrequenz 16Mhz

$sim

'*********************** Port Konfiguration ***********************************
Config Portc.3 = Input                                      '### Buzz 1
Config Portc.4 = Input                                      '### Buzz 2
Config Portc.5 = Input                                      '### Buzz 3
Config Portc.0 = Input                                      '### Master 1 (Nächste Frage)
Config Portd.2 = Input                                      '### Master 2 (Punkte +1 als Interrupt Service)
Config Portd.3 = Input                                      '### Master 3 (Punkte -1 als Interrupt Service)
Config Porta.0 = Output                                     '### Buzz 1 LED
Config Porta.1 = Output                                     '### Buzz 2 LED
Config Porta.2 = Output                                     '### Buzz 3 LED


'********************** Interrupts und Timer **********************************
'Interrupt 0 Punkte plus 1
On Int0 Isr_pkt_plus1                                       '### INT0(PD.2) löst isr_punkte_plus1 aus.
Config Int0 = Rising                                        '### INT0 reagiert auf steigende Flanke des Spannungssignals(bessere Abfrage)
Enable Int0
'Interrupt 2 Punkte minus 1
On Int1 Isr_pkt_minus1                                      '### INT1(PD.3) löst isr_punkte_minus1 aus.
Config Int1 = Rising                                        '### INT1 reagiert auf steigende Flanke des Spannungssignals(bessere Abfrage)
Enable Int1
'Timer Für Zufallszahlen
Config Timer2 = Timer , Prescale = 1024                     '### Timer2 zählt ca 61mal/Sekunden bis 255. Dieser Wert wird vom Programm dazu
                                                             'genutzt unterschiedliche ___rseed Werte zu generieren(rseed=timer2)->richtige Zufallszahlen

'********************** LCD Initialisierung ***********************************
Config Lcdpin = Pin , Db4 = Portd.4 , Db5 = Portd.5 , Db6 = Portd.6 , Db7 = Portd.7 , E = Portb.3 , Rs = Portb.0
                  '### Konfiguration der Datenleitungen im 4Bit Modus; RS=Registerauswahl und E=Enable)

Config Lcd = 16 * 2                                         '#### Festlegung der Zeilen und Zeichen des LCD´s (Zeichen * Zeile)

Cursor Off                                                  '### Schaltet das Blinks des Cursors ab


'*********************Variablendeklaration*************************************
Dim Buzz As Byte                                            '### Buzz gibt an welcher Buzz gedrückt wurde: Buzz=1>Buzz 1;Buzz=2>Buzz 2

Dim Buz1pkt As Integer                                      '### Zum Angeben des Punktestandes. Je Nachdem welchen Wert BuzzXpunkte hat wird Eine Andere Zahl
Dim Buz2pkt As Integer                                      ' ausgegeben. Also LCD = "Buzz 1 hat " ; Buz1pkt ; " Punkte!"
Dim Buz3pkt As Integer

Dim Nr_1 As Byte                                            '### Nr_1 und 2 sind die Zahlen zum Rechnen Byte(0-255) da nur 0-100 Zahlen verwendet werden
Dim Nr_2 As Byte
Dim Antwort As Integer                                      '### Integer wegen möglichen Minuszahlen als Antwort

Dim Plus_minu As String * 5                                 '### String reicht für 254(oder 255) Buchstaben "* x" gibt an wie viele Buchstaben benötigt werden
                                                              'also wie viele Bytes der String maximal braucht. Variable wird benutzt für Rechnung
                                                              '.(if plus_minu = "mal" then Zahl1 * Zahl2) und für die Operatoranzeige am LCD

Dim Wich_op As Byte                                         '### Byte da Wich_op nur 1-3 annehmen kann. Damit wird plus_minu Variable bestimmt.
                                                             'If wich_op = 1 then plus_minu = "mal"

Dim ___rseed As Byte                                        '#### Für bessere zufallszahlen  ___rseed gibt an ab wo er anfängt zu zählen und so erzeugt das
                                                             'Programm nicht immer die gleichen Zufallszahlen. Byte weil Int2 = 0-255 und INT2 = ___rseed

Dim Right_ As Bit                                           '### Zum abfragen ob eine Frage right oder falsch beantwortet wurde und dann verschieden Text
                                                             'anzuzeigen. Bit da es nur 1=right oder 0=falsch sein kann.

Dim Blinks As Byte                                          '### Zum hochzählen der For-Schleife (damit z.B. die LED(von Buzzer 1)5 mal im 1s Takt blinkt)

Dim End_intr As Bit                                         '### Um aus dem Loop zu springen nach Abarbeitung der Interrupts und dem return
'**********************Variableninitialisierung********************************
Buz1pkt = 0
Buz2pkt = 0
Buz3pkt = 0




'##############################################################################
'##############################################################################
'########################-STARTROUTINE, NICHT DAS SPIEL-#######################
'##############################################################################
Cls
Waitms 1000                                                 '### Damit das Programm nach Stromanschluss nicht sofort anfängt und das Display sich erstmal hochfährt
   Locate 1 , 1                                             '### Cursor auf die erste Zeile/spalte
   Lcd "AmSiRe Buzz"                                        '### Text anzeigen , der Text bleibt dort bis er überschrieben
Waitms 200                                                  'oder das Display gecleart wird
   Locate 2 , 2                                             '### Springt in die Mitte der Zeile 2
   Lcd "loading..."                                         '### Symbolische Spielerei
Waitms 2000                                                 '### 2 Sekunden warten
   Locate 2 , 1                                             '### Sringt auf Zeile 2 Anfang um sie gleich komplett zu überschreiben
   Lcd "Buzzs detected"                                     '### Überschreibt "loading"
Waitms 800
   Locate 2 , 1                                             '### Das gleiche wie oben nochmal
   Lcd "Le 3 player mode"
Waitms 800
   Cls                                                      '### Bildschirm ist kurz leer
Waitms 400
   Locate 1 , 1
   Lcd "NEXT QUESTION to"
   Locate 2 , 1
   Lcd "start the game!"


'##############################-LOOP FÜR SPIELBEGIN-###########################
Do
nop                                                         '### Endlossschleife in der Nichts passiert. Dient nur als Startknopf
Loop Until Pinc.0 = 0                                       '### Sprung aus der Schleife wenn Master 1 gedrückt wird(Nächste Frage)


'#################################-Spielstart-#################################
Cls
   Locate 1 , 6
   Lcd "Have fun"
Waitms 1000
   Locate 1 , 1
   Lcd "Greetz Amadeus"
   Locate 2 , 1
   Lcd "Silas und Rene"
Waitms 1500
   Cls
   Locate 1 , 5
   Lcd "LETSE GO :)"





'##############################################################################
'########################-HAUPTPROGRAMM- DAS SPIEL#############################
'##############################################################################

'#########################-BEGIN NÄCHSTE FRAGE-################################
Next_quest:                                                 '### Hier springt das Programm hin wenn man Nächste Frage drückt


'######################Reset wegen letzter Runde###############################
Portc.3 = 1
Portc.4 = 1                                                 '### LED der Buzz leuchtet standardmäßig, ist aus nach Zeitablauf zum beantworten.
Portc.5 = 1
___rseed = ___rseed + Timer2                                '### Setzen des rseed Wertes anhand des internen Timers für bessere Zufallszahlen.
'End_intr = 0                                                '### Wurde im letzten Durchlauf benutzt um aus dem Loop zu springen ach ISR(Punkte+-1)-Hat an dieser Stelle nicht funktioniert

'######Berechnung Zahl1/2 und Operator(Div,Mult,Sub,Add) für LCD und Antwort###
Nr_1 = Rnd(100)                                             '### Zahl 1 wird eine zufällige Zahl zwischen 0 und 100 zugeteilt.
Nr_2 = Rnd(100)                                             '### Zahl 2 siehe Zahl 1
Wich_op = Rnd(4)                                            '### Variable "wich_op" wird eine Zahl zwischen 0 und 3 zugeiteilt.
'Wich_op = Rnd(5)

If Wich_op = 1 Then                                         '### Je nachdem welche Zahl der Variable "wich_op" zugeteilt wurde weisst die If-Funktion "plus_minu"
      Plus_minu = "minus"                                   ' einen String zu. Dieser wird benutzt um auf dem LCD "plus"/"minus"/"geteilt"/"mal" darzustellen.
   Elseif Wich_op = 2 Then
      Plus_minu = "plus"
   Elseif Wich_op = 3 Then
      Plus_minu = "times"
'   Elseif Wich_op = 4 Then
'      Plus_minu = "durch"
End If

If Plus_minu = "minus" Then                                 '### Je nachdem welcher String in der Varialbe "plus_minu" steht
      Antwort = Nr_1 - Nr_2                                 'Wird der Variable Antwort ein Wert zugewiesen welcher sich aus Nr_1 und Nr_2
   Elseif Plus_minu = "plus" Then                           'zusammensetzt. Ist plus_minu z.B. "mal" wird Nr_1 * Nr_2 genommen
      Antwort = Nr_1 + Nr_2
   Elseif Plus_minu = "times" Then
      Antwort = Nr_1 * Nr_2
'   Elseif Plus_minu = "durch" Then
'      Antwort = Nr_1 / Nr_2
End If

'############################Fragenanzeige#####################################
Cls                                                         '### Bildschirm löschen von letzter Frage/Punktestand etc
   Locate 1 , 1
   Lcd "3"
Waitms 1000
   Locate 1 , 1
   Lcd "2"
Waitms 1000
   Locate 1 , 1
   Lcd "1"                                                  '### Countdown 3, 2,1 ' Frage
Waitms 1000                                                 '### 3 sekunden Zeit bis zur nächsten Frage
   Locate 1 , 1
   Lcd "Calculate:"                                         '### Anzeige der Frage beginnt hier
   Locate 2 , 1
   Lcd Nr_1 ; " " ; Plus_minu ; " " ; Nr_2                  '### Anzeige Variable Nr_1(1-99) ; Variable "plus_minu"(plus,mal,minus) ; Variable Nr_2




'##############################################################################
'############################BUZZER ABFRAGEN###################################
'#Sprung mit GOTO nach "Buzz_dran" -> Anfang des Blinklichts für beantwortung##


'##################Alle Buzzer abfragen-Wer drückt erster?#####################
Do
   If Pinc.3 = 0 Then                                       '### Bei Drücken von Buzzer 1 geht es hier weiter
         Buzz = 1                                           '### Setzen der Variable die bestimmt welcher Buzz dran ist(wird öfterer benutzt)
         Goto Buzz_dran                                     '### Goto um aus dem LOOP sofort raus zu springen, die anderen Buzzer werden nicht mehr abgefragt
      Elseif Pinc.4 = 0 Then
         Buzz = 2
         Goto Buzz_dran                                     '### Buzzer 2 Weiterleitung
      Elseif Pinc.5 = 0 Then
         Buzz = 3
         Goto Buzz_dran                                     '### Buzzer 3 Weiterleitung
      Elseif Pinc.0 = 0 Then
         Exit Do                                            '### Wenn Keiner die Antwort weiß kann man mit "master Nächste Frage"
   End If                                                   'aus der schleife Springen
Loop


'######Nächste Frage weil zu schwere Frage(Wenn jemand Master 1 drückt)########
'Waitms 1000
'Cls                                                         '### Frage wird gelöscht
'   Locate 1 , 1
'   Lcd "Haha was it"
'   Locate 2 , 1                                             '### Kurzer Text, das die Frage zu schwer war
'   Lcd "too comlicated?"
'Waitms 2000
'Cls
'         Locate 1 , 1
'         Lcd "Na gut dann die"                              '### Wurde aus Platzgründen raus genommen
'         Locate 2 , 1
'         Lcd "nächste Frage!"
'         Waitms 1000
Goto Next_quest                                             '### Spiel geht bei next_quest: wieder weiter(Sprung)


'#################Blinklicht für den Buzzer der dran ist#######################
Buzz_dran:                                                  '### GOTO aus der Buzzer Abfrage
Cls                                                         '### Frage verschwindet
   Locate 1 , 1
If Buzz = 1 Then                                            '### Wenn die Variable Buzz = 1 ist zeige Buzzer 1 in Zeile 1,1 an
      Lcd "Buzzer 1 you"
   Elseif Buzz = 2 Then                                     '### Wenn die Variable Buzz = 2 ist zeige Buzzer 2 in Zeile 1,1 an
      Lcd "Buzzer 2 you"
   Elseif Buzz = 3 Then                                     '### Wenn die Variable Buzz = 3 ist zeige Buzzer 3 in Zeile 1,1 an
      Lcd "Buzzer 3 you"
End If




'##############################################################################
'#################Das Blinklicht als mehrere For-Schleifen#####################
'##############################5 Sekunden######################################


   Locate 2 , 1
   Lcd "have 5 Seconds!"

For Blinks = 0 To 5 Step 1                                  '### For-Schleife(Führt etwas X mal aus also 0 to 5 = 5 mal) und zählt dabei eine Variable
   If Buzz = 1 Then                                         'immer 1 hoch(step 1)) In diesem Fall die Variable Blinks(Blinker)
         Toggle Porta.0
      Elseif Buzz = 2 Then                                  '### Je nachdem welchen Wert die Variable Buzz hat, blinkt eine andere LED(toggle)
         Toggle Porta.1                                     'z.B. Buzz = 2 dann blinkt die LED von Buzzer 2(1 mal je durchgang)
      Elseif Buzz = 3 Then
         Toggle Porta.2
   End If
   Waitms 200                                               '### 5*200=1000= 1 Sekunde
Next                                                        '### Nächster Durchgang und einmal Blinks + 1 hochzählen

'##############################4 Sekunden######################################
'Blinks = 0
   Locate 2 , 1
   Lcd "have 4 seconds!"                                    '### Anzeige wie viel Sekunden übrig sind , Überschreibt "have 5 Seconds"

For Blinks = 0 To 6 Step 1                                  '### Siehe Erste For-Schleife , gleiche Funktion nur wird diese 6 mal ausgeführt(6*167 = ca. 1000)
   If Buzz = 1 Then
         Toggle Porta.0
      Elseif Buzz = 2 Then                                  '### Je nachdem welchen Wert die Variable Buzz hat, blinkt eine andere LED(toggle)
         Toggle Porta.1
      Elseif Buzz = 3 Then
         Toggle Porta.2
   End If
   Waitms 167                                               '### 6*167=1000= 1 Sekunde
Next                                                        '### Siehe oben

'##############################3 Sekunden######################################
'Blinks = 0
   Locate 2 , 1
   Lcd "have 3 seconds!"                                    '### Anzeige wie viel Sekunden übrig sind , Überschreibt "have 4 Seconds"

For Blinks = 0 To 8 Step 1                                  '### Siehe Erste For-Schleife , gleiche Funktion nur wird diese 8 mal ausgeführt(8*125 = 1000)
   If Buzz = 1 Then
         Toggle Porta.0
      Elseif Buzz = 2 Then                                  '### Je nachdem welchen Wert die Variable Buzz hat, blinkt eine andere LED(toggle)
         Toggle Porta.1
      Elseif Buzz = 3 Then
         Toggle Porta.2
   End If
   Waitms 125                                               '### 8*125=1000
Next                                                        '### Siehe oben

'##############################2 Sekunden######################################
'Blinks = 0
   Locate 2 , 1
   Lcd "have 2 seconds!!"                                   '### Anzeige wie viel Sekunden übrig sind , Überschreibt "have 3 Seconds"

For Blinks = 0 To 10 Step 1                                 '### Siehe Erste For-Schleife , gleiche Funktion nur wird diese 8 mal ausgeführt(8*125 = 1000)
   If Buzz = 1 Then
         Toggle Porta.0
      Elseif Buzz = 2 Then                                  '### Je nachdem welchen Wert die Variable Buzz hat, blinkt eine andere LED(toggle)
         Toggle Porta.1
      Elseif Buzz = 3 Then
         Toggle Porta.2
   End If
   Waitms 100                                               '### 10*100 = 1000
Next

'##############################1 Sekunden######################################
'Blinks = 0
   Locate 2 , 1
   Lcd "have 1 second!!!"                                   '### Anzeige wie viel Sekunden übrig sind , Überschreibt "have 2 Seconds"

For Blinks = 0 To 7 Step 1                                  '### Siehe Erste For-Schleife , gleiche Funktion nur wird diese 7 mal ausgeführt(7*75 = 500)
   If Buzz = 1 Then
         Toggle Porta.0
      Elseif Buzz = 2 Then                                  '### Je nachdem welchen Wert die Variable Buzz hat, blinkt eine andere LED(toggle)
         Toggle Porta.1
      Elseif Buzz = 3 Then
         Toggle Porta.2
   End If
   Waitms 75                                                '### 7*75=500 = 0.5 Sekunden
Next

'##############################0.5 Sekunden####################################
'Blinks = 0
   Locate 2 , 1
   Lcd "are out of time!"                                   '### Anzeige wie viel Sekunden übrig sind , Überschreibt "have 1 Seconds"

For Blinks = 0 To 13 Step 1                                 '### Siehe Erste For-Schleife , gleiche Funktion nur wird diese 13 mal ausgeführt(13*125 = 1000)
   If Buzz = 1 Then
         Toggle Porta.0
      Elseif Buzz = 2 Then                                  '### Je nachdem welchen Wert die Variable Buzz hat, blinkt eine andere LED(toggle)
         Toggle Porta.1
      Elseif Buzz = 3 Then
         Toggle Porta.2
   End If
   Waitms 38                                                '### 13*38=500 = 0.5 Sekunden
Next                                                        '### Led sollte aus sein, ENDE des Blinkens




'##############################################################################
'##ZEIT ABGELAUFEN - Anzeige Antwort.Weiter gehts durch richtig/Falsch-Drücken#
'#########Interrupts anschalten für richtig/falsch + Punktezählen##############


Enable Interrupts                                           '### Interrupts werden eingeschaltet(right/Falsch Buzzer) zur Weiterleitung
End_intr = 0                                                '### Wurde im letzten Durchlauf benutzt um aus dem Loop zu springen(Right or wrong?) nach ISR(Punkte+-1)

'###############War es richtig oder falsch? Spieler muss drücken###############
Cls                                                         '### Löscht "Buzzer 1 you are out of time"
Locate 1 , 4
Lcd "STOP YOUR"                                             '### LCD Anzeige das die Zeit abgelaufen ist
   Locate 2 , 3
   Lcd "TIME IS OVER"
Waitms 2000
Cls
   Locate 1 , 1
   Lcd "The answer is"
   Locate 2 , 6
   Lcd Antwort                                              '### Ausgabe der Antwort (Oben berechnet)
Waitms 3000
   Locate 1 , 1
   Lcd "Push right or"                                      '### Für die Leute die nicht verstehen das man falsch/right drücken soll
   Locate 2 , 1
   Lcd "wrong! " ; Antwort                                  '### Damit Antwort immernoch angezeigt wird hier nochmal "Antwort"
Do
   nop                                                      '### Schleife in der nichts passiert bis right oder falsch gedrückt wurde
Loop Until End_intr = 1

'##############################################################################
'#########Anzeige ob richtig oder falsch und Anzeige des Punktestandes#########
'#################Interrupts ausschalten für richtig/Falsch####################

Disable Interrupts                                          '### Interrupts werden ausgeschaltet damit die Knöpfe right/falsch außer Funktion sind


'#################LCD Anzeige dass es richtig oder falsch war##################
Cls                                                         '### Löscht "Push right or wrong!   " aus dem LCD
If Right_ = 1 Then                                          '### Wenn die Variable right_ = 1 ist schreibe "Correct;)" in das LCD
      Locate 1 , 1
      Lcd "Correct;)"
   Waitms 2000
Else                                                        '### Wenn die Variable right_ nicht 1 ist, schreibe "Uh..wrong" in das LCD
      Locate 1 , 1
      Lcd "Uh..wrong!"
   Waitms 2000
End If


'################Punktestand anzeigen und neue Frage anfangen##################
Cls
   Locate 1 , 1
   Lcd "Points Buz1: " ; Buz1pkt                            '### Anzeige der Punkte,Text in"" dann mithilfe von den aktuellen Punktestand aus der Variablen
   Locate 2 , 1
   Lcd "Points Buz2: " ; Buz2pkt                            '### Anzeige der Punkte,Text in"" dann mithilfe von den aktuellen Punktestand aus der Variablen

Waitms 2000
Cls
   Locate 1 , 1
   Lcd "Points Buz3: " ; Buz3pkt                            '### Anzeige Punkte von Buzzer 3 ,wegen 2 Zeilen Display zeigt erst 1/2 and dann 3

Do                                                          '### Do Loop schleife bis "Nächste Frage" gedrückt wird
      Locate 2 , 1
      Lcd "Push Next Quest"                                 '### LCD zeigt im Wechsel "Pust Next Quest" und "             "(Space16)
   Waitms 600                                               '### Für das Blinken von "push next quest" wartet er hier 0.6s->Es wird 6 Sek angezeigt
      Locate 2 , 1
      Lcd Space(16)                                         '### Die Anzeige blinkt hier ein/aus deswegen das Space
   Waitms 300                                               '### "push next quest" ist 0.3 sek ausgeblendet
Loop Until Pinc.0 = 0                                       '### Wird "Nächste Frage" gedrückt dann endet die Schleife

Waitms 1000                                                 '### Er wartet 1s bis es von vorne los geht
Goto Next_quest                                             '### Sprung zum Anfang des Spiels

End


'##############################################################################
'############################INTERRUPT SERIVICES###############################


'######Interrupt 1#######
Isr_pkt_plus1:
Right_ = 1
If Buzz = 1 Then                                            '###Abfrage welcher Buzz gedrückt hatte Bei Fragestellung
      Buz1pkt = Buz1pkt + 1                                 'Punkte Buzz1 werden hochgezählt
   Elseif Buzz = 2 Then
      Buz2pkt = Buz2pkt + 1                                 'Punkte Buzz2 werden hochgezählt
   Elseif Buzz = 3 Then
      Buz3pkt = Buz3pkt + 1                                 'Punkte Buzz3 werden hochgezählt
End If

End_intr = 1                                                '### Um gleich nach dem Return aus dem Loop zu kommen und bei der LCD Punkteanzeige weiter zu machen
Return


'######Interrupt 2#######
Isr_pkt_minus1:
Right_ = 0
If Buzz = 1 Then                                            '###Abfrage welcher Buzz gedrückt hatte Bei Fragestellung
      Buz1pkt = Buz1pkt - 1                                 'Punkte Buzz1 werden hochgezählt
   Elseif Buzz = 2 Then
      Buz2pkt = Buz2pkt - 1                                 'Punkte Buzz2 werden hochgezählt
   Elseif Buzz = 3 Then
      Buz3pkt = Buz3pkt - 1                                 'Punkte Buzz3 werden hochgezählt
End If

End_intr = 1                                                '### Um gleich nach dem Return aus dem Loop zu kommen und bei der LCD Punkteanzeige weiter zu machen
Return
