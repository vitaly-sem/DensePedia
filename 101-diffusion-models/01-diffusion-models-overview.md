# Diffusion Models — Overview

---

## Что такое Diffusion Models

**Факт:** Diffusion Models — класс generative AI моделей, которые **постепенно удаляют шум** из случайного шума, восстанавливая изображение/видео.

```
Forward (training):
Изображение → Добавляем шум → Полный шум

Reverse (generation):
Случайный шум → Убираем шум шаг за шагом → Изображение
```

**Факт:** Термин "Diffusion" — из термодинамики (частицы диффундируют из области высокой концентрации в низкую).

---

## Основные модели — сравнение

| Модель | Разработчик | Тип | Качество | Скорость | Open Source | API |
|---|---|---|---|---|---|---|
| **Stable Diffusion (SD)** | Stability AI | Text-to-Image, Img2Img | ⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ (Apache 2.0) | ✅ |
| **SDXL** | Stability AI | TTI (1024x1024) | ⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ | ✅ |
| **SD3 / SD3.5** | Stability AI | TTI (MMDiT) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ | ✅ |
| **Flux** | Black Forest Labs | TTI | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ (dev/schnell) | ✅ |
| **DALL-E 3** | OpenAI | TTI, Edit | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ❌ | ✅ (ChatGPT) |
| **Midjourney** | Midjourney | TTI | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ❌ | ❌ (Discord) |
| **Imagen (2/3)** | Google | TTI | ⭐⭐⭐⭐ | ⭐⭐⭐ | ❌ | ❌ (Vertex AI) |
| **Firefly** | Adobe | TTI, Edit, Fill | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ❌ | ✅ |
| **Sora** | OpenAI | **Text-to-Video** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ❌ | Limited |
| **Runway Gen-3** | Runway | Text-to-Video | ⭐⭐⭐⭐ | ⭐⭐⭐ | ❌ | ✅ |

---

## Stable Diffusion — детали

**Факты:**
- **Open Source** (Apache 2.0) — можно запускать локально (GPU 6GB+)
- Работает в **Latent Space** (VAE кодирует изображение в сжатое представление)
- Компоненты: UNet (denoising), CLIP (text encoder), VAE (latent/image)

```python
# Stable Diffusion — генерация (diffusers library)
from diffusers import StableDiffusionXLPipeline
import torch

pipe = StableDiffusionXLPipeline.from_pretrained(
    "stabilityai/stable-diffusion-xl-base-1.0",
    torch_dtype=torch.float16
).to("cuda")

# Text-to-Image
image = pipe(
    prompt="a cat wearing a spacesuit, digital art",
    negative_prompt="blurry, low quality",
    width=1024,
    height=1024,
    num_inference_steps=30,       # Quality vs speed (20-50)
    guidance_scale=7.5,           # Adherence to prompt
    seed=42                       # Reproducibility
).images[0]

image.save("cat_spacesuit.png")

# Image-to-Image
from diffusers import StableDiffusionXLImg2ImgPipeline
pipe = StableDiffusionXLImg2ImgPipeline.from_pretrained(...)

image = pipe(
    prompt="turn this photo into an oil painting",
    image=init_image,
    strength=0.7  # How much to transform (0-1)
).images[0]
```

**Факт:** `guidance_scale` (CFG) — баланс между креативностью (низкое) и точностью следования промпту (высокое).

**Факт:** `seed` фиксирует случайность — с одним seed и промптом результат всегда одинаковый.

---

## Flux — новейшая open-source модель

**Факты:**
- Разработчик: **Black Forest Labs** (бывшие сотрудники Stability AI)
- **Flux.1 Pro** — коммерческая
- **Flux.1 Dev** — open-weight (для некоммерческого)
- **Flux.1 Schnell** — open-weight, быстрая (4 шага вместо 28)
- Качество: конкурирует с Midjourney/DALL-E 3
- Поддержка текста на изображении (лучше всех)

---

## DALL-E 3

**Факты:**
- Встроен в **ChatGPT Plus** (через естественный язык)
- Превосходное **понимание сложных промптов** (natural language)
- Ограничения: unsafe content (violence, hate), no NSFW
- Макс. разрешение: 1792x1024

```
Prompt: "A photorealistic close-up of a raindrop on a green leaf
with a tiny reflection of a rainbow in the drop, macro photography,
f/2.8 aperture, soft morning light"
```

---

## Midjourney

**Факты:**
- Только через **Discord** (самый популярный интерфейс)
- **Стиль**: художественный, кинематографичный (узнаваемый "MJ look")
- Параметры: `--ar 16:9`, `--style raw`, `--v 6.1`, `--s 250`
- **Inpainting / Outpainting**: Vary Region, Pan

```
/imagine prompt: cyberpunk city street at night, neon lights,
rain, cinematic lighting, photorealistic, 8k --ar 16:9 --v 6.1
```

---

## Text-to-Video — Sora

**Факты:**
- **Sora (OpenAI)** — генерация видео до 60 сек, физика, multi-shot
- **Runway Gen-3** — текст→видео, Film Grain, Motion Brush
- **Pika Labs** — текст→видео, редактирование видео
- **Stable Video Diffusion** — изображение→видео (open source)

| Модель | Длительность | Качество | Реализм | Custom |
|---|---|---|---|---|
| Sora | До 60 сек | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ❌ |
| Runway Gen-3 | До 10 сек | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ |
| Pika 2.0 | До 3 сек | ⭐⭐⭐ | ⭐⭐⭐ | ✅ |

---

## Ограничения Diffusion Models

| Ограничение | Описание | Пример |
|---|---|---|
| **Hands/Fingers** | Анатомические ошибки | 6 пальцев |
| **Текст** | Плохо читается (кроме Flux) | Буквы размыты |
| **Faces** | Искажения лиц (особенно группы) | Разные глаза |
| **Consistency** | Персонаж меняется между кадрами | Разная одежда |
| **Physics** | Нарушение законов физики | Парящие объекты |
| **Copyright** | Модель может воспроизвести стиль | Стиль Disney/Marvel |
| **Bias** | Гендерные/расовые стереотипы | CEO = white male |
| **Safety** | Deepfake, NSFW, violence | Модерация vary |

---

## Использование в .NET

```csharp
// Вызов Stable Diffusion API из .NET
using var http = new HttpClient();

var request = new
{
    prompt = "a cat in space",
    negative_prompt = "blurry, low quality",
    width = 1024,
    height = 1024,
    steps = 30
};

var response = await http.PostAsJsonAsync(
    "http://localhost:7860/sdapi/v1/txt2img", request);

var result = await response.Content.ReadFromJsonAsync<StableDiffusionResult>();
var imageBytes = Convert.FromBase64String(result.Images[0]);

await File.WriteAllBytesAsync("output.png", imageBytes);
```

---

## Чек-лист

- [ ] Stable Diffusion: open-source, latent space, UNet + CLIP + VAE
- [ ] Flux: новейшая open-source, конкурирует с Midjourney
- [ ] DALL-E 3: natural language, ChatGPT integration
- [ ] Midjourney: Discord, artistic style, параметры --ar --v --s
- [ ] Sora: text-to-video, Runway, Pika
- [ ] Ограничения: hands, text, consistency, physics, bias
- [ ] .NET: HTTP API to SD WebUI / Replicate / Stability AI API
