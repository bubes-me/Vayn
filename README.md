# VAYN — Model Management Platform

Zero-cost serverless architecture for talent agency management.

## Stack

| Layer | Service | Cost |
|---|---|---|
| Frontend | React SPA → GitHub Pages | $0 |
| Auth | Amazon Cognito (≤10k MAU) | $0 |
| Compute | AWS Lambda + API Gateway | $0 |
| Database | DynamoDB (25GB Always Free) | $0 |
| Images | Google Drive API (15GB) | $0 |

---

## Project Structure

```
vayn-platform/
├── src/
│   ├── main.jsx              ← App root, Amplify config, view router
│   ├── index.css             ← Tailwind directives
│   └── components/
│       ├── AuthView.jsx      ← Cognito login screen
│       ├── DirectoryGrid.jsx ← Public model roster with filter sliders
│       └── Dashboard.jsx     ← Role-gated internal dashboard
├── lambda/
│   └── index.mjs             ← Lambda handler (list / create / update models)
└── public/
    └── assets/
        └── vayn-logo.svg     ← Place your logo here
```

---

## Frontend Setup

### 1. Install dependencies

```bash
npm create vite@latest vayn-platform -- --template react
cd vayn-platform
npm install aws-amplify
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### 2. Configure Tailwind

`tailwind.config.js`:
```js
export default {
  content: ["./index.html", "./src/**/*.{js,jsx}"],
  theme: { extend: {} },
  plugins: [],
};
```

`src/index.css`:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 3. Fill in AWS resource IDs

Open `src/main.jsx` and update `VAYN_CONFIG`:

```js
const VAYN_CONFIG = {
  cognitoUserPoolId:       "us-east-1_XXXXXXXXX",
  cognitoUserPoolClientId: "XXXXXXXXXXXXXXXXXXXXXXXXXX",
  cognitoRegion:           "us-east-1",
  apiBase: "https://XXXXXXXXXX.execute-api.us-east-1.amazonaws.com/prod",
};
```

### 4. Build & deploy to GitHub Pages

```bash
npm run build
# Push dist/ to gh-pages branch, or use:
npm install -D gh-pages
npx gh-pages -d dist
```

---

## Lambda Deployment

### 1. Create the function

```bash
cd lambda
zip -r function.zip index.mjs
aws lambda create-function \
  --function-name vayn-models-api \
  --runtime nodejs20.x \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::ACCOUNT_ID:role/vayn-lambda-role \
  --environment Variables="{TABLE_NAME=VaynModels}"
```

### 2. IAM permissions for the Lambda role

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:Scan",
    "dynamodb:PutItem",
    "dynamodb:UpdateItem",
    "dynamodb:GetItem"
  ],
  "Resource": "arn:aws:dynamodb:REGION:ACCOUNT_ID:table/VaynModels"
}
```

### 3. API Gateway setup

Create a REST API with:
- `GET  /models`          → Lambda (Cognito Authorizer optional for public view)
- `POST /models`          → Lambda + Cognito Authorizer
- `PUT  /models/{modelId}`→ Lambda + Cognito Authorizer
- Enable CORS on all resources

---

## DynamoDB Table

**Table name:** `VaynModels`
**Partition key:** `modelId` (String)

Attributes used:
- `name` (String)
- `status` (String) — `Active` | `On Hold` | `Pending Review`
- `height_cm`, `bust_cm`, `waist_cm`, `hips_cm` (Number)
- `profileImageId` (String) — Google Drive file ID
- `createdAt`, `updatedAt` (String, ISO timestamps)

---

## Cognito Groups

Create these groups in your User Pool and assign users accordingly:

| Group | Access |
|---|---|
| `Admin` | Full roster management |
| `Talent Manager` | Full roster management |
| `Scout` | Write-only talent submission |

---

## Google Drive Image Setup

1. Upload model photos to a shared Drive folder.
2. For each photo, right-click → **Share** → **Anyone with the link** → **Viewer**.
3. Copy the file ID from the URL: `drive.google.com/file/d/**FILE_ID**/view`
4. Store that FILE_ID as `profileImageId` in DynamoDB.

The frontend generates thumbnail URLs automatically:
```
https://drive.google.com/thumbnail?id=FILE_ID&sz=w400
```

No Drive API key required for public thumbnails.

---

## View Routing

Hash-based routing (no server config needed for GitHub Pages):

| Hash | View |
|---|---|
| `#app` (default) | Login or Dashboard depending on auth state |
| `#directory` | Public-facing model directory (no login required) |

Link the public directory from your agency website:
```html
<a href="https://yourorg.github.io/vayn-platform/#directory">View Talent</a>
```
