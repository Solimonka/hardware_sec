# Немного теории: как устроена случайность в OpenTitan

В OpenTitan случайность проходит по такой цепочке:

```text
ENTROPY_SRC  ──►  CSRNG  ──►  EDN  ──►  аппаратные потребители
  (физический      (DRBG,       (раздача          (AES, KMAC,
    шум)          криптостойкий)  энтропии)          OTBN ...)
                                  │
                                  └──►  прошивка (через DIF)
```

1. ENTROPY_SRC собирает шум из нескольких кольцевых генераторов. Это настоящая физическая случайность, у которой нет "сида" — её нельзя воспроизвести. Дальше энтропия проходит проверки здоровья и кондиционирование.
2. CSRNG берёт энтропию от ENTROPY_SRC как затравочный материал и на её основе генерирует криптографически стойкие псевдослучайные байты. Работает быстро и много, но по сути уже детерминирован после сида.
3. EDN — это не генератор сам по себе, а транспорт: он забирает готовые случайные биты из CSRNG и раздаёт их аппаратным блокам, а также может отдать их прошивке по запросу.

Если нужна истинная случайность — идём в ENTROPY_SRC. Если нужен криптостойкий и быстрый поток байтов — идём в CSRNG. Если нужно просто проверить что-то в симуляции — проще всего программный вариант.

## Способ 1. Аппаратный ГСЧ через ENTROPY_SRC (истинная энтропия)

Самый "честный" способ. ENTROPY_SRC выдаёт биты, рождённые физическим шумом, поэтому результат нельзя предсказать или воспроизвести.

Через библиотеку DIF (Device Interface Functions) — заголовок sw/device/lib/dif/dif_entropy_src.h.

В репозитории уже есть готовый smoke-тест, на который удобно ориентироваться:

```bash
bazel test //sw/device/tests:entropy_src_smoketest_sim_verilator
```

Общая логика работы с ним выглядит так:

```c
#include "sw/device/lib/dif/dif_entropy_src.h"
#include "sw/device/lib/base/mmio.h"
#include "sw/device/lib/testing/test_framework/ottf_main.h"
#include "hw/top_earlgrey/sw/autogen/top_earlgrey.h"
 
OTTF_DEFINE_TEST_CONFIG();
 
static dif_entropy_src_t entropy_src;
 
bool test_main(void) {
  mmio_region_t base =
      mmio_region_from_addr(TOP_EARLGREY_ENTROPY_SRC_BASE_ADDR);
  CHECK_DIF_OK(dif_entropy_src_init(base, &entropy_src));
 
  /* Конфигурируем: включаем FIPS-режим и маршрутизацию в прошивку */
  dif_entropy_src_config_t config = {
      .fips_enable = true,
      .route_to_firmware = true,
  };
  CHECK_DIF_OK(dif_entropy_src_configure(&entropy_src, config,
                                          kDifToggleEnabled));
 
  /* Читаем слово истинной энтропии */
  uint32_t word = 0;
  CHECK_DIF_OK(dif_entropy_src_read(&entropy_src, &word));
  LOG_INFO("true entropy word = 0x%08x", word);
 
  return true;
}
```

Точные названия полей структуры конфигурации в разных версиях OpenTitan могут немного отличаться. Самый надёжный ориентир — готовый файл sw/device/tests/entropy_src_smoketest.c: откройте его и скопируйте актуальный порядок инициализации.

В симуляции физического шума нет, поэтому ENTROPY_SRC в Verilator работает с подменённым (детерминированным) источником или вовсе требует специальной настройки. Именно поэтому в работе с mbedTLS мы обошли его программной заглушкой. На реальном железе этот способ — основной.

## Способ 2. Криптостойкий ДСЧ через CSRNG

CSRNG — аппаратный DRBG (NIST SP 800-90A, CTR_DRBG). Он "сидится" энтропией из ENTROPY_SRC и после этого быстро выдаёт криптографически стойкие байты. Это баланс между скоростью и стойкостью, поэтому на практике именно CSRNG — рабочий генератор для ключей, сессий и nonce.

Через sw/device/lib/dif/dif_csrng.h. Готовый пример — sw/device/tests/csrng_smoketest.c:

```bash
bazel test //sw/device/tests:csrng_smoketest_sim_verilator
```

Общая схема:

```c
#include "sw/device/lib/dif/dif_csrng.h"
#include "sw/device/lib/base/mmio.h"
#include "sw/device/lib/testing/test_framework/ottf_main.h"
#include "hw/top_earlgrey/sw/autogen/top_earlgrey.h"
 
OTTF_DEFINE_TEST_CONFIG();
 
static dif_csrng_t csrng;
 
bool test_main(void) {
  mmio_region_t base =
      mmio_region_from_addr(TOP_EARLGREY_CSRNG_BASE_ADDR);
  CHECK_DIF_OK(dif_csrng_init(base, &csrng));
 
  /* Instantiate — загрузка затравочного материала (энтропия из ENTROPY_SRC) */
  dif_csrng_seed_material_t seed_material = {0};
  CHECK_DIF_OK(dif_csrng_instantiate(&csrng, seed_material, &config));
 
  /* Генерируем 4 слова случайных данных */
  uint32_t data[4];
  CHECK_DIF_OK(dif_csrng_generate(&csrng, &seed_material, data, 4));
 
  for (int i = 0; i < 4; i++) {
    LOG_INFO("csrng[%d] = 0x%08x", i, data[i]);
  }
 
  return true;
}
```

CSRNG выдаёт криптостойкую псевдослучайность, а не "чистый" физический шум — то есть после одинакового сида последовательность воспроизводима. Для практической безопасности это нормально, потому что сид берётся из настоящего ENTROPY_SRC.

## Способ 3. Раздача энтропии через EDN

EDN (Entropy Distribution Network) сам ничего "не придумывает" — он забирает готовые случайные биты из CSRNG и раздаёт их аппаратным потребителям, а также может отдавать прошивке. Это способ получить случайность "как это делают остальные блоки чипа".

Через sw/device/lib/dif/dif_edn.h, готовый пример — sw/device/tests/edn_smoketest.c:

```bash
bazel test //sw/device/tests:edn_smoketest_sim_verilator
```

Общая схема:

```c
#include "sw/device/lib/dif/dif_edn.h"
#include "sw/device/lib/base/mmio.h"
#include "sw/device/lib/testing/test_framework/ottf_main.h"
#include "hw/top_earlgrey/sw/autogen/top_earlgrey.h"
 
OTTF_DEFINE_TEST_CONFIG();
 
static dif_edn_t edn;
 
bool test_main(void) {
  mmio_region_t base =
      mmio_region_from_addr(TOP_EARLGREY_EDN0_BASE_ADDR);
  CHECK_DIF_OK(dif_edn_init(base, &edn));
 
  dif_edn_auto_params_t params = {
      .mode = kDifEdnAutoModeCsrng,
  };
  CHECK_DIF_OK(dif_edn_set_auto_mode(&edn, params));
 
  /* Запрашиваем 4 слова и ждём готовности */
  CHECK_DIF_OK(dif_edn_generate_start(&edn, 4));
 
  uint32_t data[4];
  CHECK_DIF_OK(dif_edn_generate_end(&edn, data));
 
  return true;
}
```

Когда нужно встроиться в штатный механизм раздачи энтропии чипа или когда требуется, чтобы случайные данные приходили по тому же пути, что и к аппаратным блокам.

## Способ 4. Программный DRBG на mbedTLS (то, что мы уже сделали)

Мы уже собрали mbedTLS и использовали mbedtls_ctr_drbg — это программный криптостойкий генератор, такой же по принципу, как CSRNG, только исполняется в коде, а не в железе. В текущем варианте источником энтропии служит программная заглушка fake_entropy_source, потому что в Verilator нет физического шума.

Удобнее работать на уровне кода (например, нужны ключи внутри прошивки и уже есть mbedTLS), а не гонять аппаратные блоки напрямую.

## Способ 5. Простой программный PRNG — LCG (линейный конгруэнтный генератор)

Формула: x_{n+1} = (a * x_n + c) mod m. Никакой криптостойкости — только для отладки, демо и быстрых проверок.

Код:

```c
static uint32_t lcg_state = 0x12345678u;
 
uint32_t lcg_rand(void) {
    /* Классические константы из glibc (для 32 бит) */
    lcg_state = lcg_state * 1103515245u + 12345u;
    return lcg_state;
}
```

## Способ 6. Простой программный PRNG — LFSR (регистр сдвига с обратной связью)

Линейный регистр сдвига с обратной связью. Быстрый, аппаратно-дружелюбный, но тоже детерминированный и не криптостойкий. Удобен, когда нужен "случайный" тестовый поток с минимумом кода.

Код:

```c
static uint32_t lfsr_state = 0xACE1u;
 
uint32_t lfsr_rand(void) {
    /* Отводы 0,2,3,5 (полином) — дают максимальный период для 32 бит */
    uint32_t bit = ((lfsr_state >> 0) ^ (lfsr_state >> 2) ^
                    (lfsr_state >> 3) ^ (lfsr_state >> 5)) & 1u;
    lfsr_state = (lfsr_state >> 1) | (bit << 31);
    return lfsr_state;
}
```

## Способ 7. Программный DRBG на HMAC/SHA-256 (альтернатива CTR_DRBG)

В mbedTLS кроме CTR_DRBG есть и другие DRBG. Один из популярных — HMAC_DRBG (на базе HMAC-SHA256). Иногда его удобнее использовать по требованиям заказчика или стандарта.

Что нужно включить в `config.h`:

```c
#define MBEDTLS_HMAC_DRBG_C
#define MBEDTLS_MD_C
#define MBEDTLS_SHA256_C
```

Код:

```c
#include "mbedtls/hmac_drbg.h"
#include "mbedtls/md.h"
 
static mbedtls_hmac_drbg_context hmac_drbg;
 
bool init_hmac_drbg(void) {
    mbedtls_hmac_drbg_init(&hmac_drbg);
 
    const mbedtls_md_info_t *md = mbedtls_md_info_from_type(MBEDTLS_MD_SHA256);
    const unsigned char pers[] = "OpenTitan_HMAC_DRBG";
 
    int ret = mbedtls_hmac_drbg_seed(&hmac_drbg, md,
                                     mbedtls_entropy_func, &entropy,
                                     pers, sizeof(pers) - 1);
    return (ret == 0);
}
 
void generate_random(uint8_t *out, size_t len) {
    mbedtls_hmac_drbg_random(&hmac_drbg, out, len);
}
```