import cv2
import pytesseract

# Load image
img = cv2.imread("receipt.png")

# Convert to grayscale
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

# Thresholding (binarization)
_, thresh = cv2.threshold(gray, 150, 255, cv2.THRESH_BINARY)

# Noise removal
denoised = cv2.medianBlur(thresh, 3)

# Optional: Deskew (if tilted)
coords = cv2.findNonZero(denoised)
angle = cv2.minAreaRect(coords)[-1]
if angle < -45:
    angle = -(90 + angle)
else:
    angle = -angle
(h, w) = denoised.shape[:2]
M = cv2.getRotationMatrix2D((w // 2, h // 2), angle, 1.0)
deskewed = cv2.warpAffine(denoised, M, (w, h),
                          flags=cv2.INTER_CUBIC, borderMode=cv2.BORDER_REPLICATE)

# OCR with Tesseract
text = pytesseract.image_to_string(deskewed, lang="eng")
print(text)
