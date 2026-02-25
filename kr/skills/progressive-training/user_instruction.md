## User Instruction
- 패지키 관리는 uv를 활용하세요.
- 이미지 처리에는 pillow 라이브러리를 활용하세요.
- 학습 프레임워크는 pytorch-lightning을 활용하세요.
- 데이터가 resize는 되어 있는데, bucket padding이 안 되어 있으니 적용하는 부분이 필요합니다.
- W_BUCKETS의 최대값을 넘는 이미지는 제외하여 배치를 구성하세요. (데이터셋 구성 시 이를 처리하는 로직이 있으나 안 되어 있을 경우 대비)
- 병렬처리 라이브러리를 활용하거나 옵션을 사용할 경우 worker수는 4로 설정해 작업하세요.
- Augmentation 참고 코드
```python
import random

class LineAugmenter:
    """
    Augmentations for screenshot line OCR (grayscale, H fixed).
    All ops keep H unchanged and keep width unchanged (we resample back to original width).
    """
    def __init__(self, seed: int = 42):
        self.rng = random.Random(seed)

    def __call__(self, img: np.ndarray) -> np.ndarray:
        # img: uint8 grayscale [H,W], H=64 fixed
        assert img.ndim == 2
        h, w = img.shape

        # ---- Photometric
        if self.rng.random() < 0.7:
            img = self._brightness_contrast(img)
        if self.rng.random() < 0.3:
            img = self._gamma(img)

        # ---- Degradation
        if self.rng.random() < 0.25:
            img = self._jpeg_artifact(img)
        if self.rng.random() < 0.25:
            img = self._blur(img)
        if self.rng.random() < 0.15:
            img = self._unsharp(img)

        # ---- UI artifacts
        if self.rng.random() < 0.20:
            img = self._highlight_bar(img)
        if self.rng.random() < 0.20:
            img = self._underline(img)
        if self.rng.random() < 0.25:
            img = self._occlusion(img)

        # ---- Tiny geometry jitter (keep width)
        if self.rng.random() < 0.20:
            img = self._x_scale_jitter_keep_width(img)

        # Final clamp
        img = np.clip(img, 0, 255).astype(np.uint8)
        assert img.shape == (h, w)
        return img

    def _brightness_contrast(self, img: np.ndarray) -> np.ndarray:
        # img float32 for safe math
        x = img.astype(np.float32)
        # contrast in [0.75, 1.25], brightness shift in [-20, 20]
        c = self.rng.uniform(0.75, 1.25)
        b = self.rng.uniform(-20, 20)
        x = x * c + b
        return np.clip(x, 0, 255).astype(np.uint8)

    def _gamma(self, img: np.ndarray) -> np.ndarray:
        gamma = self.rng.uniform(0.8, 1.25)
        x = img.astype(np.float32) / 255.0
        x = np.power(x, gamma) * 255.0
        return np.clip(x, 0, 255).astype(np.uint8)

    def _jpeg_artifact(self, img: np.ndarray) -> np.ndarray:
        # encode/decode with random quality
        q = self.rng.randint(25, 85)
        encode_param = [int(cv2.IMWRITE_JPEG_QUALITY), q]
        ok, enc = cv2.imencode(".jpg", img, encode_param)
        if not ok:
            return img
        dec = cv2.imdecode(enc, cv2.IMREAD_GRAYSCALE)
        return dec if dec is not None else img

    def _blur(self, img: np.ndarray) -> np.ndarray:
        mode = self.rng.choice(["gauss", "motion"])
        if mode == "gauss":
            k = self.rng.choice([3, 5])
            return cv2.GaussianBlur(img, (k, k), sigmaX=0)
        else:
            # simple horizontal motion blur
            k = self.rng.choice([3, 5, 7])
            kernel = np.zeros((k, k), dtype=np.float32)
            kernel[k // 2, :] = 1.0 / k
            return cv2.filter2D(img, -1, kernel)

    def _unsharp(self, img: np.ndarray) -> np.ndarray:
        # unsharp mask: img + a*(img - blur)
        blur = cv2.GaussianBlur(img, (3, 3), 0)
        a = self.rng.uniform(0.3, 0.8)
        x = img.astype(np.float32) + a * (img.astype(np.float32) - blur.astype(np.float32))
        return np.clip(x, 0, 255).astype(np.uint8)

    def _highlight_bar(self, img: np.ndarray) -> np.ndarray:
        # add a light gray rectangle background (like selection highlight)
        h, w = img.shape
        bar_h = self.rng.randint(max(3, h // 6), max(6, h // 3))
        y0 = self.rng.randint(0, max(0, h - bar_h))
        alpha = self.rng.uniform(0.10, 0.25)  # blend strength
        # highlight color: light gray (200~245)
        color = self.rng.randint(200, 245)
        overlay = img.copy().astype(np.float32)
        overlay[y0:y0 + bar_h, :] = color
        out = (1 - alpha) * img.astype(np.float32) + alpha * overlay
        return np.clip(out, 0, 255).astype(np.uint8)

    def _underline(self, img: np.ndarray) -> np.ndarray:
        # draw 1px underline at random y (common in links)
        h, w = img.shape
        y = self.rng.randint(int(h * 0.6), h - 1)
        thickness = 1
        col = self.rng.randint(0, 80)  # dark line
        out = img.copy()
        cv2.line(out, (0, y), (w - 1, y), color=int(col), thickness=thickness)
        return out

    def _occlusion(self, img: np.ndarray) -> np.ndarray:
        # small rectangle occlusion (cursor/icon/tooltips)
        h, w = img.shape
        out = img.copy()
        rect_w = self.rng.randint(max(3, w // 40), max(8, w // 10))
        rect_h = self.rng.randint(max(3, h // 6), max(10, h // 2))
        x0 = self.rng.randint(0, max(0, w - rect_w))
        y0 = self.rng.randint(0, max(0, h - rect_h))
        # occlusion can be light or dark
        fill = self.rng.randint(0, 255)
        out[y0:y0 + rect_h, x0:x0 + rect_w] = fill
        return out

    def _x_scale_jitter_keep_width(self, img: np.ndarray) -> np.ndarray:
        # simulate minor horizontal resampling artifacts without changing final W
        h, w = img.shape
        scale = self.rng.uniform(0.90, 1.10)
        new_w = max(1, int(round(w * scale)))
        tmp = cv2.resize(img, (new_w, h), interpolation=cv2.INTER_CUBIC)
        # resample back to original width
        out = cv2.resize(tmp, (w, h), interpolation=cv2.INTER_CUBIC)
        return out
```