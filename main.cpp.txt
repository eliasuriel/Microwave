#include "mbed.h"
#include <MKL25Z4.h>
#include <string>

#define RS 0x04 /* PTA1 mask */
#define RW 0x10 /* PTA4 mask */
#define EN 0x20 /* PTA5 mask */

char keypad_getkey(void);
void keypad_init(void);

void LCD_command(unsigned char command);
void LCD_data(unsigned char data);
void LCD_init(void);

void interrupt(void);
void interrupt2(void);

void motor(int value);
void turnoff(void);

void delayMs(int n);
void delayUs(int n);

int getNum(unsigned char key);

InterruptIn ir(PTA1);
InterruptIn mq2(PTA13);
PwmOut servo(PTE21);
DigitalOut Led_ex(PTE5);

int global_var = 0, global_lcd=0;

int main(void) {
    //Lineas para inicializar el TMP0
    SIM->SCGC6 |= 0x01000000; // enable clock to TPM0
    SIM->SOPT2 |= 0x01000000; // use 32.768 kHz clock
    TPM0->SC = 0; // disable timer while configuring
    TPM0->SC = 0x02; // prescaler 4
    TPM0->MOD = 0x2000; // modulo 8192
    TPM0->SC |= 0x80; // clear TOF
    TPM0->SC |= 0x08; // enable timer free-running mode
    //Aquí acaba el TMP0
    keypad_init();
    LCD_init();

    ir.rise(interrupt);
    mq2.rise(interrupt2);
    while(1){
        Led_ex=0;
        global_lcd=1;
        LCD_command(1); /* clear display */
        delayMs(500);
        LCD_command(0x80); /* set cursor at first line */
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data('P');
        LCD_data('U'); /* write the word */
        LCD_data('T');
        LCD_data(' ');
        LCD_data('T');
        LCD_data('I');
        LCD_data('M');
        LCD_data('E');
        LCD_data(':');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
        delayMs(500);

        unsigned char key1;
        unsigned char key2;
        unsigned char key3;
        unsigned char key4;
        int sel = 0;
        int sel2 = 0;
        int sel3 = 0;
        int sel4 = 0;
        int status = 0;
        int value = 0;

        std::string sel1_str;

        while(1){
            key1 = keypad_getkey();
            if (key1 == 1){
                sel = 1;
            } else 
            if(key1 == 2){
                sel = 2;
            } else 
            if(key1 == 3){
                sel = 3;
            }else
            if (key1 == 4){
                status=1;
                sel2 = 100;
                sel = 30;
            }else
            if (key1 == 5){
                sel = 4;
            } else 
            if(key1 == 6){
                sel = 5;
            } else 
            if(key1 == 7){
                sel = 6;
            }else
            if (key1 == 8){
                status=1;
                sel2 = 100;
                sel = 60;
            }else
            if (key1 == 9){
                sel = 7;
            } else 
            if(key1 == 10){
                sel = 8;
            } else
            if(key1 == 11){
                sel = 9;
            } else
            if(key1==12){
                status=1;
                sel2 = 100;
                sel = 90;
            } else
            if(key1==16){
                NVIC_SystemReset();
            }

            if (sel!=0){
                break;
            }
        }
        global_lcd=0;
        delayMs(500);

        if (status == 1){
            sel2 = 100;
        } else {
            LCD_command(1);
            LCD_command(0x80); //Set cursor at first line

            LCD_data(' ');
            LCD_data(' ');
            LCD_data(' ');
            LCD_data(' ');
            LCD_data(' ');
            LCD_data('0');
            LCD_data('0');
            LCD_data(':');
            LCD_data('0');
            sel1_str = std::to_string(sel);
            LCD_data(static_cast<unsigned char>(sel1_str[0]));
            sel2 = getNum(key2);
        }
        
        if (sel2 != 100){
            LCD_command(1); /* clear display */
            delayMs(500);
            LCD_command(0x80); /* set cursor at first line */
            LCD_data(' ');
            LCD_data(' ');
            LCD_data(' ');
            LCD_data(' ');
            LCD_data(' ');
            LCD_data('0');
            LCD_data('0');
            LCD_data(':');
            std::string sel2_str = std::to_string(sel2);
            LCD_data(static_cast<unsigned char>(sel1_str[0]));
            LCD_data(static_cast<unsigned char>(sel2_str[0]));
            delayMs(500);
            sel3=getNum(key3);
            if (sel3!=100){
                LCD_command(1); /* clear display */
                delayMs(500);
                LCD_command(0x80); /* set cursor at first line */
                LCD_data(' ');
                LCD_data(' ');
                LCD_data(' ');
                LCD_data(' ');
                LCD_data(' ');
                LCD_data('0');
                LCD_data(static_cast<unsigned char>(sel1_str[0]));
                LCD_data(':');
                LCD_data(static_cast<unsigned char>(sel2_str[0]));
                std::string sel3_str = std::to_string(sel3);
                LCD_data(static_cast<unsigned char>(sel3_str[0]));
                sel4=getNum(key4);
                if(sel4!=100){
                    LCD_command(1); /* clear display */
                    delayMs(500);
                    LCD_command(0x80); /* set cursor at first line */
                    LCD_data(' ');
                    LCD_data(' ');
                    LCD_data(' ');
                    LCD_data(' ');
                    LCD_data(' ');
                    LCD_data(static_cast<unsigned char>(sel1_str[0]));
                    LCD_data(static_cast<unsigned char>(sel2_str[0]));
                    LCD_data(':');
                    LCD_data(static_cast<unsigned char>(sel3_str[0]));
                    std::string sel4_str = std::to_string(sel4);
                    LCD_data(static_cast<unsigned char>(sel4_str[0]));
                }
            }
        }

        if (sel2 == 100){
            value = sel;
        }
        else if(sel3 == 100){
            value = (sel*10)+sel2;
        }
        else if(sel4 == 100){
            value = (sel*60)+(sel2*10)+sel3;
        }
        else{
            value = ((sel*60)*(10))+(sel2*60)+(sel3*10)+sel4;
        }

        delayMs(100);

        global_var=1;
        motor(value);
        global_var=0;
    }
}

int getNum(unsigned char key2){
    int sel2 = 0;
    while(1){
        key2 = keypad_getkey();
        if (key2 == 14){
            sel2 = 10;
        } else 
        if (key2 == 1){
            sel2 = 1;
        } else 
        if (key2 == 2){
            sel2 = 2;
        } else 
        if (key2 == 3){
            sel2 = 3;
        }else
        if (key2 == 5){
            sel2 = 4;
        } else 
        if (key2 == 6){
            sel2 = 5;
        } else 
        if (key2 == 7){
            sel2 = 6;
        }else
        if (key2 == 9){
            sel2 = 7;
        } else 
        if (key2 == 10){
            sel2 = 8;
        }else 
        if (key2 == 11){
            sel2 = 9;
        }else 
        if(key2==13 || key2==15){
            sel2 = 100;
        }else
        if(key2==16){
            NVIC_SystemReset();
        }

        if (sel2!=0){
            if (sel2==10) {
                sel2=0;
            }
            break;
        }
    }
    return sel2;
}

void turnoff(void){
    if(global_var==1){
        servo.pulsewidth(0.000); // servo position determined by a pulsewidth between 1-2ms
    }
}

void motor(int value){
    int min,sec,sel_test;
    int d1, u1, d, u;
    while(value>=0){
        Led_ex=1;

        sel_test = keypad_getkey();
        if (sel_test==16){
            NVIC_SystemReset();
        }
        LCD_command(1); //Clear
        min = value/60;
        sec = value%60;
        d = sec/10;
        u = sec%10;
        d1 = min/10;
        u1 = min%10;
        std::string temp_u = std::to_string(u);
        std::string temp_d = std::to_string(d);
        std::string temp_u1 = std::to_string(u1);
        std::string temp_d1 = std::to_string(d1);
        unsigned char u_Test;
        u_Test = static_cast<unsigned char>(temp_u[0]);
        unsigned char d_Test;
        d_Test = static_cast<unsigned char>(temp_d[0]);
        unsigned char u1_Test;
        u1_Test = static_cast<unsigned char>(temp_u1[0]);
        unsigned char d1_Test;
        d1_Test = static_cast<unsigned char>(temp_d1[0]);
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(d1_Test);
        LCD_data(u1_Test);
        LCD_data(':');
        LCD_data(d_Test);
        LCD_data(u_Test);
        servo.pulsewidth(0.002); // servo position determined by a pulsewidth between 1-2ms
        value=value-1;
        delayMs(1000);
    } 
    int i=0;
    turnoff();
    while(i<=3){
        delayMs(250);
        Led_ex=1;
        delayMs(250);
        Led_ex=0;
        i++;
    }
}

void interrupt(){
    if(global_var==1){
        servo.pulsewidth(0.000); // servo position determined by a pulsewidth between 1-2ms
    }
    Led_ex=1;
    LCD_command(1); /* clear display */
    while(ir){
        LCD_command(0x80); /* set cursor at first line */
        LCD_data('C');
        LCD_data('l');
        LCD_data('o');
        LCD_data('s');
        LCD_data('e');
        LCD_data(' ');
        LCD_data('d'); /* write the word */
        LCD_data('o');
        LCD_data('o');
        LCD_data('r');
        LCD_data(' ');
        LCD_data('t');
        LCD_data('o');
        LCD_command(0xC0);
        LCD_data('c');
        LCD_data('o');
        LCD_data('n');
        LCD_data('t');
        LCD_data('i');
        LCD_data('n');
        LCD_data('u');
        LCD_data('e');
        delayMs(500);
    }
    Led_ex=0;
    LCD_command(1); /* clear display */
    LCD_command(0x80); /* set cursor at first line */
    if(global_var==1){
        servo.pulsewidth(0.002); // servo position determined by a pulsewidth between 1-2ms
    }
    if(global_lcd){
        LCD_command(1); /* clear display */
        delayMs(500);
        LCD_command(0x80); /* set cursor at first line */
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data('P');
        LCD_data('U'); /* write the word */
        LCD_data('T');
        LCD_data(' ');
        LCD_data('T');
        LCD_data('I');
        LCD_data('M');
        LCD_data('E');
        LCD_data(':');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
    }
}

void interrupt2(){
    if(global_var==1){
        servo.pulsewidth(0.000); // servo position determined by a pulsewidth between 1-2ms
    }
    LCD_command(1); /* clear display */
    while(mq2==0){
        LCD_command(0x80); /* set cursor at first line */
        LCD_data('S');
        LCD_data('m');
        LCD_data('o');
        LCD_data('k');
        LCD_data('e');
        LCD_data(' ');
        LCD_data('d'); /* write the word */
        LCD_data('e');
        LCD_data('t');
        LCD_data('e');
        LCD_data('c');
        LCD_data('t');
        LCD_data('e');
        LCD_data('d');
        LCD_command(0xC0);
        LCD_data('c');
        LCD_data('h');
        LCD_data('e');
        LCD_data('c');
        LCD_data('k');
        LCD_data(' ');
        LCD_data('d');
        LCD_data('o');
        LCD_data('o');
        LCD_data('r');
        delayMs(500);
    }
    LCD_command(1); /* clear display */
    LCD_command(0x80); /* set cursor at first line */
    if(global_var==1){
        servo.pulsewidth(0.002); // servo position determined by a pulsewidth between 1-2ms
    }
    if(global_lcd){
        LCD_command(1); /* clear display */
        delayMs(500);
        LCD_command(0x80); /* set cursor at first line */
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data('P');
        LCD_data('U'); /* write the word */
        LCD_data('T');
        LCD_data(' ');
        LCD_data('T');
        LCD_data('I');
        LCD_data('M');
        LCD_data('E');
        LCD_data(':');
        LCD_data(' ');
        LCD_data(' ');
        LCD_data(' ');
    }
}

void LCD_init(void){
    SIM->SCGC5 |= 0x1000; /* enable clock to Port D */
    PORTD->PCR[0] = 0x100; /* make PTD0 pin as GPIO */
    PORTD->PCR[1] = 0x100; /* make PTD1 pin as GPIO */
    PORTD->PCR[2] = 0x100; /* make PTD2 pin as GPIO */
    PORTD->PCR[3] = 0x100; /* make PTD3 pin as GPIO */
    PORTD->PCR[4] = 0x100; /* make PTD4 pin as GPIO */
    PORTD->PCR[5] = 0x100; /* make PTD5 pin as GPIO */
    PORTD->PCR[6] = 0x100; /* make PTD6 pin as GPIO */
    PORTD->PCR[7] = 0x100; /* make PTD7 pin as GPIO */
    PTD->PDDR = 0xFF; /* make PTD7-0 as output pins */
    SIM->SCGC5 |= 0x0200; /* enable clock to Port A */
    PORTA->PCR[2] = 0x100; /* make PTA2 pin as GPIO */
    PORTA->PCR[4] = 0x100; /* make PTA4 pin as GPIO */
    PORTA->PCR[5] = 0x100; /* make PTA5 pin as GPIO */
    PTA->PDDR |= 0x34; /* make PTA5, 4, 2 as out pins*/
    delayMs(30); /* initialization sequence */
    LCD_command(0x38);
    delayMs(1);
    LCD_command(0x01);
    /* set 8-bit data, 2-line, 5x7 font */
    LCD_command(0x38);
    /* move cursor right */
    LCD_command(0x06); 
    /* clear screen, move cursor to home */
    LCD_command(0x01);
    /* turn on display, cursor blinking */
    LCD_command(0x0F);
}

void LCD_command(unsigned char command){
    PTA->PCOR = RS | RW; /* RS = 0, R/W = 0 */
    PTD->PDOR = command;
    PTA->PSOR = EN; /* pulse E */
    delayMs(0);
    PTA->PCOR = EN; 
    if (command < 4)
    delayMs(4); /* command 1 and 2 needs up to 1.64ms */
    else
    delayMs(1); /* all others 40 us */
}

void LCD_data(unsigned char data){
    PTA->PSOR = RS; /* RS = 1, R/W = 0 */
    PTA->PCOR = RW;
    PTD->PDOR = data;
    PTA->PSOR = EN; /* pulse E */
    delayMs(0);
    PTA->PCOR = EN;
    delayMs(1);
}

void delayMs(int n){
    int i;
    for (i = 0; i<n;i++){
        while((TPM0->SC & 0x80) == 0) { } // wait until the TOF is set
        TPM0->SC |= 0x80; // clear TOF
    }
}

void delayUs(int n){
    int i;
    for (i = 0; i<n;i++){
        while((TPM0->SC & 0x80) == 0) { } // wait until the TOF is set
        TPM0->SC |= 0x80; // clear TOF
    }
}

char keypad_getkey(void) {
    int row, col;
    const char row_select[] = {0x01, 0x02, 0x04, 0x08}; 
    /* one row is active */
    /* check to see any key pressed */

    PTC->PDDR |= 0x0F; /* enable all rows */
    PTC->PCOR = 0x0F;
    delayUs(2); /* wait for signal return */
    col = PTC-> PDIR & 0xF0; /* read all columns */
    PTC->PDDR = 0; /* disable all rows */
    if (col == 0xF0)
    return 0; /* no key pressed */

    /* If a key is pressed, we need find out which key.*/ 
    for (row = 0; row < 4; row++)
    { PTC->PDDR = 0; /* disable all rows */

    PTC->PDDR |= row_select[row]; /* enable one row */
    PTC->PCOR = row_select[row]; /* drive active row low*/

    delayUs(2); /* wait for signal to settle */
    col = PTC->PDIR & 0xF0; /* read all columns */

    if (col != 0xF0) break; 
    /* if one of the input is low, some key is pressed. */
    }

    PTC->PDDR = 0; /* disable all rows */

    if (row == 4)
    return 0; /* if we get here, no key is pressed */

    /* gets here when one of the rows has key pressed*/ 
    /*check which column it is*/

    if (col == 0xE0) return row * 4 + 1; /* key in column 0 */
    if (col == 0xD0) return row * 4 + 2; /* key in column 1 */
    if (col == 0xB0) return row * 4 + 3; /* key in column 2 */
    if (col == 0x70) return row * 4 + 4; /* key in column 3 */
    return 0; /* just to be safe */
}

void keypad_init(void){
    SIM->SCGC5 |= 0x0800;  /* enable clock to Port C */
    PORTC->PCR[0] = 0x103; /* PTD0, GPIO, enable pullup*/
    PORTC->PCR[1] = 0x103; /* PTD1, GPIO, enable pullup*/
    PORTC->PCR[2] = 0x103; /* PTD2, GPIO, enable pullup*/
    PORTC->PCR[3] = 0x103; /* PTD3, GPIO, enable pullup*/
    PORTC->PCR[4] = 0x103; /* PTD4, GPIO, enable pullup*/
    PORTC->PCR[5] = 0x103; /* PTD5, GPIO, enable pullup*/
    PORTC->PCR[6] = 0x103; /* PTD6, GPIO, enable pullup*/
    PORTC->PCR[7] = 0x103; /* PTD7, GPIO, enable pullup*/
    PTC->PDDR = 0x0F; /* make PTD7-0 as input pins */
}
