## Made by V.V.Poliakov for WWSERVICE. 

## Code is in ../code.txt and ../src/main.cpp

## Used PlatformIO ( https://platformio.org , https://github.com/platformio )
Some questionable moments in this code:

    1. Why so many global variables, and almost none local?

    The dynamic memory of Arduino is quickly runs away, causing troubles. And, I noticed that the main reason of it
    is unoptimized usage of local variables, new ones are creating but old ones is not clearing up as soon as needed.
    The most easy way to fix it - using global variables, and rewrite it when needed, even if local one preffered.
    
    2. Why used "Error" against other words, like "Warning"?

    Word "Error" is very negative, and stimulates people more that something like "Warning" or "Trouble". 
    For example, even in compiler the "Warnings" not stop the code against "Errors".

    3. Why default settings are on this level?

    They are temporary, for testing. Also they allowed by PZEM-004t

    4. Where is the scheme of this device? 

    The scheme is not done, because this is only basical side-version script, specified for shield.
    The testing details are:
    PZEM-004t
    Atmel Atmega328pb (Arduino Uno analog)
    D1ROBOT LCD SHIELD 
    Wires
    Relay(SRD-05VDC-SL-C)  --- to pin D2
    Just contact to apply 5V on it, used to control power supply --- to pin D3
    To simulate supply control, used button connected directly between D3 and 5V, with GND through resistor.

    5. How to contact me to ask some questions:

    Phone number: +380 50 851 79 06
    E-mail: poliakov.vladislav.v@gmail.com
    Telegram: https://t.me/ParrKenna | @ParrKenna | Search by phone opened
    Linkedin: https://www.linkedin.com/in/владислав-поляков-691b28298/
    Djinni: https://djinni.co/q/dbfdf0f6b8/
    Viber by phone +380508517906
    
