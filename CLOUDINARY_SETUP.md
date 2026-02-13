# VitalConnect - Cloudinary Integration for RFID Cards

## Setup Instructions

### 1. Get Cloudinary Credentials

1. Go to [Cloudinary](https://cloudinary.com/) and sign up for a free account
2. Once logged in, go to your Dashboard
3. Copy the following values:
   - **Cloud Name**
   - **API Key**
   - **API Secret**

### 2. Update Backend Environment Variables

Edit `/backend/.env` and add your Cloudinary credentials:

```env
# Cloudinary (Image Storage)
CLOUDINARY_CLOUD_NAME=your_cloud_name_here
CLOUDINARY_API_KEY=your_api_key_here
CLOUDINARY_API_SECRET=your_api_secret_here
```

Replace the placeholder values with your actual Cloudinary credentials from step 1.

### 3. Restart Backend Server

After updating the `.env` file, restart your backend server:

```bash
cd backend
npm run start:dev
```

## Features

- **Insurance Card Upload**: Users can upload insurance card images (JPEG, PNG, WebP)
- **QR Code Upload**: Support for uploading QR code images
- **Automatic Optimization**: Images are automatically optimized by Cloudinary
- **Size Validation**: Max 5MB file size
- **Type Validation**: Only allows image formats (JPEG, PNG, WebP)
- **Secure Storage**: Images stored on Cloudinary CDN with secure URLs

## API Endpoints

### Upload Insurance Card

```
POST /api/v1/rfid/upload/insurance
Content-Type: multipart/form-data

Body: { file: <image file> }

Response: {
  url: "https://res.cloudinary.com/...",
  publicId: "vitalconnect/insurance-cards/...",
  width: 1000,
  height: 800,
  format: "jpg"
}
```

### Upload QR Code

```
POST /api/v1/rfid/upload/qr
Content-Type: multipart/form-data

Body: { file: <image file> }

Response: {
  url: "https://res.cloudinary.com/...",
  publicId: "vitalconnect/qr-codes/...",
  width: 500,
  height: 500,
  format: "png"
}
```

## Usage in Web UI

When creating or editing an RFID card:

1. Click "Upload insurance card photo or QR code"
2. Select an image file (max 5MB)
3. The image is automatically uploaded to Cloudinary
4. The Cloudinary URL is saved in the database
5. The image is displayed using the Cloudinary CDN URL

## Database Schema Changes

The following columns were updated in the `rfid_cards` table:

- `insuranceImage` → `insuranceImageUrl` (stores Cloudinary URL)
- `qrCodeImage` → `qrCodeImageUrl` (stores Cloudinary URL)

## Benefits of Cloudinary

1. **Performance**: Images served from Cloudinary's global CDN
2. **Optimization**: Automatic compression and format conversion
3. **Scalability**: No storage limits on your server
4. **Reliability**: 99.9% uptime guarantee
5. **Transformation**: Can resize/crop images on-the-fly via URL parameters

## Example Cloudinary URL

```
https://res.cloudinary.com/your_cloud_name/image/upload/v1234567890/vitalconnect/insurance-cards/abc123.jpg
```

You can transform images by adding parameters:

- Resize: `/w_500,h_300/`
- Quality: `/q_auto/`
- Format: `/f_webp/`

Example: `https://res.cloudinary.com/.../w_500,h_300,q_auto,f_webp/vitalconnect/insurance-cards/abc123.jpg`
