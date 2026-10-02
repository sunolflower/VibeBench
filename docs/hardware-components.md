# Компоненти VibeBench

Лещенко Ольга Сергіївна, ІО-31.

## Поточна конфігурація ЛР1

Джерелом фактичних налаштувань є `firmware/Core/Src/main.c`, `stm32c0xx_hal_msp.c`, `firmware/Core/Inc/main.h`, `firmware/firmware.ioc`, `firmware/Makefile`, `diagram.json` і `wokwi.toml`. Файли шаблону збережено без змін.

| Вузол | Фактичне налаштування | Джерело для апаратного зіставлення |
|---|---|---|
| MCU | STM32C031C6, Cortex-M0+, HSI 48 МГц, без FPU; компілятор `-mcpu=cortex-m0plus -mthumb`, linker `STM32C031C6Tx_FLASH.ld`. C6 має 32 КБ Flash і 12 КБ RAM. | [ST DS13867 Rev 4, с. 1](https://www.st.com/resource/en/datasheet/stm32c031c6.pdf#page=1), [модель Wokwi](https://docs.wokwi.com/parts/board-st-nucleo-c031c6) |
| USART2 | PA2 TX, PA3 RX, AF1, push-pull, без pull; 115200 бод, 8 біт, без парності, 1 стоп-біт, TX/RX, без flow control. `__io_putchar` викликає блокуючий HAL_UART_Transmit. | [DS13867, с. 32, 35, таблиці 12–13](https://www.st.com/resource/en/datasheet/stm32c031c6.pdf#page=32); [UM2953 Rev 2, с. 16, таблиця 8](https://www.st.com/resource/en/user_manual/um2953-stm32-nucleo64-board-mb1717-stmicroelectronics.pdf#page=16) |
| LED | PA5, GPIO output push-pull, без pull, VERY_HIGH speed, початковий SET. Перемикання, друк tick і HAL_Delay 1000 мс у циклі. | [UM2953, с. 16, §6.10](https://www.st.com/resource/en/user_manual/um2953-stm32-nucleo64-board-mb1717-stmicroelectronics.pdf#page=16), LD4 USER активний високим рівнем |
| I²C1 | PB8 SCL, PB9 SDA, AF6, open-drain, NOPULL, LOW speed; джерело PCLK1 48 МГц, Timing=0x20303E5D, адресація 7 біт, analog filter enabled, digital filter=0. Ініціалізація є, транзакцій до датчика немає. | [DS13867, с. 34, 37, таблиці 12–13](https://www.st.com/resource/en/datasheet/stm32c031c6.pdf#page=34) |
| PC13 | User_Button, EXTI rising, NOPULL. У шаблоні немає логіки запуску вікна кнопкою. | [UM2953, с. 16, §6.11](https://www.st.com/resource/en/user_manual/um2953-stm32-nucleo64-board-mb1717-stmicroelectronics.pdf#page=16) |
| ADC | Не увімкнений у .ioc; немає MX_ADC_Init та драйвера потенціометра. | [DS13867, с. 20–21, §3.14](https://www.st.com/resource/en/datasheet/stm32c031c6.pdf#page=20) описує можливості MCU, а не стан прошивки |
| Симуляція | У diagram.json одна Nucleo; serial monitor TX → PA3, RX ← PA2. firmware у TOML є HEX, elf є ELF у `firmware/build/`. | [Wokwi project config](https://docs.wokwi.com/vscode/project-config) |

Timing містить PRESC=2, SCLDEL=3, SDADEL=0, SCLH=62, SCLL=93. Саме значення регістра не є вимірюванням частоти шини. Для ЛР2 ціль 100 кГц треба перевірити CubeMX і логічним аналізатором із заданими rise/fall time та підтягуванням.

## Майбутня схема ЛР2

Це пропозиція, компонентів у diagram.json ще немає. Кнопка й потенціометр є органами керування. Для трьох інтерфейсів передбачено GPIO, I²C та ADC.

| Компонент | План підключення | Параметри й джерела |
|---|---|---|
| MPU6050, Wokwi `wokwi-mpu6050` | VCC → 3V3, GND → GND, SCL → PB8, SDA → PB9, AD0 → GND. INT, XDA/XCL не використовувати. | Адреса 0x68; AD0=VCC дає 0x69. Вхідні accelX/Y/Z у g; за замовчуванням 0,0,1. [Wokwi, Pin names / Units](https://docs.wokwi.com/parts/wokwi-mpu6050) |
| Чіп MPU6050 | Для фізичної схеми окремо звірити модуль, підтягування SCL/SDA до 3.3 В та розв'язку живлення. | VDD 2.375–3.46 В; 400 кГц максимум I²C; 16-бітне signed прискорення, ±2/4/8/16 g, 16384 LSB/g при ±2 g. Типово 0.5 мА accelerometer only, 3.9 мА accel+gyro+DMP. [PS-MPU-6000A-00 Rev 3.4, с. 10, 13–14](https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Datasheet1.pdf#page=13) |
| Потенціометр, Wokwi `wokwi-potentiometer` | VCC → 3V3, GND → GND, SIG → PA0 / ADC_IN0. | План 12 біт, raw 0–4095, вхід 0–VREF+, за прийнятого VREF+=3.3 В це 0–3.3 В. [DS13867, с. 32 та §5.3.15 ADC](https://www.st.com/resource/en/datasheet/stm32c031c6.pdf#page=32); [Wokwi, Pin names](https://docs.wokwi.com/parts/wokwi-potentiometer). Модель не відтворює повний аналоговий тракт. |
| Зовнішня кнопка, Wokwi `wokwi-pushbutton` | Контакт 1.l → PA1, 2.l → GND; GPIO input з pull-up. Зміна PC13 не потрібна. | Активний LOW, підтвердження стану 30 мс. Контакти 1.l/1.r завжди з'єднані, як і 2.l/2.r; натискання замикає групи 1 та 2. Брязкіт увімкнений за замовчуванням. [Wokwi, Pin names / Bouncing](https://docs.wokwi.com/parts/wokwi-pushbutton) |
| LED PA5 та USART2 | Зберегти піни; у ЛР2 змінити призначення LED на стан епізоду. | Показ результату й діагностики без додаткового дисплея. Фактичний шаблон зараз лише блимає. |

Для фізичного потенціометра початковий вибір 10 кОм між 3.3 В і GND. Постійний струм дільника за законом Ома близько 0.33 мА. Це інженерне припущення, конкретну модель ще не придбано. Кнопка з внутрішнім pull-up не має фіксованого паспортного струму без параметрів підтягування. Споживання модуля GY-521 залежить від його стабілізатора й LED; струм чіпа не є струмом готового модуля. Симуляція не перевіряє бюджет живлення.

## План регістрів MPU6050

| Регістр | Роль у ЛР2 | Офіційна карта RM-MPU-6000A-00 Rev 4.2 |
|---|---|---|
| WHO_AM_I 0x75 | Очікувати 0x68 незалежно від AD0; адреса шини й ідентифікатор різні поняття. | [с. 45, §4.32](https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Register-Map1.pdf#page=45) |
| PWR_MGMT_1 0x6B | Вийти зі sleep; перед читанням дочекатися стабілізації за специфікацією. | [с. 40–41, §4.28](https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Register-Map1.pdf#page=40) |
| ACCEL_CONFIG 0x1C | AFS_SEL=0, діапазон ±2 g, self-test вимкнений. | [с. 15, §4.5](https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Register-Map1.pdf#page=15) |
| CONFIG 0x1A, SMPLRT_DIV 0x19 | Узгодити внутрішній DLPF і частоту оновлення з читанням 100 Гц; звірити фактичну модель Wokwi. | [с. 11–13, §4.2–4.3](https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Register-Map1.pdf#page=11) |
| ACCEL_XOUT_H…ACCEL_ZOUT_L 0x3B–0x40 | Burst-read 6 байт, зібрати старший/молодший байти у int16_t, поділити на 16384 для g. | [с. 29, §4.17](https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Register-Map1.pdf#page=29) |

HAL I²C очікує зсунуту адресу `0x68 << 1`; це перевіряється в коментарі DevAddress функції HAL_I2C_Mem_Read у збереженому `firmware/Drivers/STM32C0xx_HAL_Driver/Src/stm32c0xx_hal_i2c.c`. Жодного такого виклику для MPU6050 у поточній прошивці немає.

## Межі достовірності та альтернативи

MPU6050 обрано через доступну модель Wokwi. ADXL345 був би достатнім акселерометром; п'єзодатчик вимагав би іншого аналогового тракту. Потенціометр перевіряє ADC, на відміну від UART-команди. Одна зовнішня кнопка дозволяє перевірити debounce, не прив'язуючи логіку до PC13 шаблону. Докладний зв'язок із вимогами є в [PRD](PRD.md).

ST DS13867 та UM2953 доступні як офіційні PDF. Параметри MPU6050 звірені з індексованими фрагментами офіційних документів TDK; пряме завантаження старих PDF-посилань 04.10.2026 перенаправляло на пошук. Повне читання PDF цього дня не підтверджене. Номери сторінок стосуються зазначених ревізій, перед фізичною реалізацією потрібен доступ до повних документів виробника. Модель Wokwi дозволяє задавати прискорення, але не доводить шум, точність, струм споживання чи механічну поведінку реального стенда.
