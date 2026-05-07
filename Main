import os
import json
import io
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from googleapiclient.http import MediaIoBaseDownload, MediaIoBaseUpload

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://vulcansticker.es"],
    allow_methods=["GET", "POST", "PUT"],
    allow_headers=["*"],
)

CALC_CONFIG_FILENAME = "vulcan-calc-config.json"
CALC_DRIVE_FOLDER_ID = "1UkhCIcOZ3yZ_wCKkGqsiaIAX9tUNg0mH"


def get_drive_service():
    creds = Credentials(
        token=None,
        refresh_token=os.environ["GOOGLE_REFRESH_TOKEN"],
        client_id=os.environ["GOOGLE_CLIENT_ID"],
        client_secret=os.environ["GOOGLE_CLIENT_SECRET"],
        token_uri="https://oauth2.googleapis.com/token"
    )
    return build("drive", "v3", credentials=creds)


def find_config_file(service):
    results = service.files().list(
        q=f"name='{CALC_CONFIG_FILENAME}' and '{CALC_DRIVE_FOLDER_ID}' in parents and trashed=false",
        fields="files(id)"
    ).execute()
    files = results.get("files", [])
    return files[0]["id"] if files else None


@app.get("/health")
async def health():
    return {"status": "ok"}


@app.get("/calc/load")
async def calc_load():
    try:
        service = get_drive_service()
        file_id = find_config_file(service)
        if not file_id:
            return JSONResponse({"config": None})
        request = service.files().get_media(fileId=file_id)
        buf = io.BytesIO()
        downloader = MediaIoBaseDownload(buf, request)
        done = False
        while not done:
            _, done = downloader.next_chunk()
        buf.seek(0)
        config = json.loads(buf.read())
        return JSONResponse({"config": config})
    except Exception as e:
        return JSONResponse({"config": None, "error": str(e)})


@app.post("/calc/save")
async def calc_save(request: Request):
    try:
        data = await request.json()
        service = get_drive_service()
        content = json.dumps(data, ensure_ascii=False).encode("utf-8")
        buf = io.BytesIO(content)
        media = MediaIoBaseUpload(buf, mimetype="application/json", resumable=False)
        file_id = find_config_file(service)
        if file_id:
            service.files().update(
                fileId=file_id,
                media_body=media
            ).execute()
        else:
            service.files().create(
                body={
                    "name": CALC_CONFIG_FILENAME,
                    "parents": [CALC_DRIVE_FOLDER_ID]
                },
                media_body=media
            ).execute()
        return JSONResponse({"status": "ok"})
    except Exception as e:
        return JSONResponse({"status": "error", "error": str(e)})
