from flask import Flask
import uuid, os
from flask_cors import CORS


def create_app():
    app = Flask(__name__)

    app.secret_key = str(uuid.uuid4())  # para deploy
    # app.secret_key = 'DEV'

    UPLOAD_FOLDER = 'uploads'
    app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
    app.config.from_mapping(
        DATABASE=os.path.join(app.instance_path, 'vivo_crc.sqlite'),
    )

    CORS(app)
    os.makedirs(UPLOAD_FOLDER, exist_ok=True)
    os.makedirs(app.instance_path, exist_ok=True)

    from . import vivo_crc
    app.register_blueprint(vivo_crc.bp)

    from . import sqlite_db
    sqlite_db.init_app(app)

    return app
