# Spotify_songs

## Uvod
U ovom projektu analizirali smo veliki skup podataka koji sadrži informacije o pjesmama sa Spotify CSV-a. Cilj je bio razumjeti i organizirati podatke kroz proces predprocesiranja, modeliranja baze podataka, punjenja dimenzijskih i fact tablica te analizirati podatke pomoću OLAP alata. Kroz ETL proces u Pentaho Data Integrationu i korištenjem SQLAlchemy biblioteke u Pythonu, izgradili smo dimenzijski model podataka koji omogućuje detaljnu analizu i izvještavanje o različitim aspektima glazbenih sadržaja, kao što su popularnost pjesama, žanrovi, albumi i vrijeme objave. Na kraju smo podatke vizualizirali u Power BI-u kako bismo prikazali ključne trendove i obrasce u glazbenim podacima te demonstrirali primjenu OLAP operacija za dubinsku analizu.

## Programske skripte
### Predprocesiranje
```python
import pandas as pd

# Učitavanje originalnog CSV-a
CSV_FILE_PATH = r"C:\Users\David\Downloads\2_VJEZBE_SPOTIFY_SONGS\spotify_songs.csv"
df = pd.read_csv(CSV_FILE_PATH)

print("CSV veličina prije obrade:", df.shape)

# Brisanje redaka s nedostajućim vrijednostima
df = df.dropna()
print("CSV veličina nakon brisanja NaN vrijednosti:", df.shape)

# Spremanje predprocesiranog skupa podataka
df.to_csv("spotify_songs_PROCESSED.csv", index=False)
print("Podaci spremljeni u spotify_songs_PROCESSED.csv")
```

Ova Python skripta koristi biblioteku pandas za obradu podataka. Prvo učitava originalnu CSV datoteku sa Spotify pjesmama. Nakon toga ispisuje veličinu podataka (broj redaka i stupaca) prije bilo kakve obrade. Zatim briše sve retke koji imaju nedostajuće vrijednosti (NaN), te ispisuje novu veličinu skupa podataka nakon čišćenja. Na kraju, očišćeni i pripremljeni skup podataka sprema u novu CSV datoteku pod imenom "spotify_songs_PROCESSED.csv".
To je osnovni korak predprocesiranja podataka kako bi se osigurala kvaliteta i spremnost podataka za daljnju analizu ili pohranu u bazu.

### Stvaranje baze podataka
```python
import pandas as pd
from sqlalchemy import create_engine, Column, String, Integer, Float, ForeignKey, Date, Table
from sqlalchemy.orm import sessionmaker, declarative_base, relationship
from datetime import datetime

# Učitavanje CSV-a
CSV_FILE_PATH = r"C:\Users\David\Desktop\SKLADIŠTE I RUDARENJE PODATAKA\spotify_songs_PROCESSED.csv"
df = pd.read_csv(CSV_FILE_PATH)

# Definicija baze
Base = declarative_base()

class Album(Base):
    __tablename__ = 'album'
    album_id = Column(String(50), primary_key=True)
    album_name = Column(String(255))
    release_date = Column(Date)

    tracks = relationship("Track", back_populates="album")

class Track(Base):
    __tablename__ = 'track'
    track_id = Column(String(50), primary_key=True)
    track_name = Column(String(255))
    track_artist = Column(String(255))
    track_popularity = Column(Integer)
    duration_ms = Column(Integer)
    danceability = Column(Float)
    energy = Column(Float)
    valence = Column(Float)
    key = Column(Integer)
    album_id = Column(String(50), ForeignKey('album.album_id'))

    album = relationship("Album", back_populates="tracks")
    playlists = relationship("Playlist", secondary="playlist_track", back_populates="tracks")

class Playlist(Base):
    __tablename__ = 'playlist'
    playlist_id = Column(String(100), primary_key=True)
    playlist_name = Column(String(255))
    playlist_genre = Column(String(100))
    playlist_subgenre = Column(String(100))

    tracks = relationship("Track", secondary="playlist_track", back_populates="playlists")

class Playlist_Track(Base):
    __tablename__ = 'playlist_track'
    track_id = Column(String(50), ForeignKey('track.track_id'), primary_key=True)
    playlist_id = Column(String(50), ForeignKey('playlist.playlist_id'), primary_key=True)

# Spajanje na MySQL bazu
engine = create_engine('mysql+pymysql://root:root@localhost:3306/spotify_songs', echo=False)
Base.metadata.drop_all(engine)
Base.metadata.create_all(engine)

Session = sessionmaker(bind=engine)
session = Session()

# Dodavanje albuma
albums = df[['track_album_id', 'track_album_name', 'track_album_release_date']].drop_duplicates()
for _, row in albums.iterrows():
    try:
        release_date = datetime.strptime(row['track_album_release_date'], "%Y-%m-%d").date()
    except ValueError:
        release_date = None  # ako ne može parsirati datum, ostavi kao NULL

    album = Album(
        album_id=row['track_album_id'],
        album_name=row['track_album_name'],
        release_date=release_date
    )
    session.add(album)
session.commit()

# Dodavanje pjesama
tracks = df.drop_duplicates(subset=['track_id'])
for _, row in tracks.iterrows():
    track = Track(
        track_id=row['track_id'],
        track_name=row['track_name'],
        track_artist=row['track_artist'],
        track_popularity=row['track_popularity'],
        duration_ms=row['duration_ms'],
        danceability=row['danceability'],
        energy=row['energy'],
        valence=row['valence'],
        key=row['key'],
        album_id=row['track_album_id']
    )
    session.add(track)
session.commit()

# Dodavanje playlista
playlists = df[['playlist_id', 'playlist_name', 'playlist_genre', 'playlist_subgenre']].drop_duplicates()
seen_ids = set()
for _, row in playlists.iterrows():
    pid = row['playlist_id']
    if pid not in seen_ids:
        playlist = Playlist(
            playlist_id=pid,
            playlist_name=row['playlist_name'],
            playlist_genre=row['playlist_genre'],
            playlist_subgenre=row['playlist_subgenre']
        )
        session.add(playlist)
        seen_ids.add(pid)
session.commit()

# Dodavanje povezanih zapisa
playlist_tracks = df[['track_id', 'playlist_id']].drop_duplicates()
for _, row in playlist_tracks.iterrows():
    pt = Playlist_Track(track_id=row['track_id'], playlist_id=row['playlist_id'])
    session.add(pt)
session.commit()

print("Baza je uspješno popunjena.")
```

Ova Python skripta koristi pandas za učitavanje predprocesiranih podataka iz CSV datoteke i SQLAlchemy za definiranje objektnog modela baze podataka te punjenje MySQL baze podataka s podacima. U skripti su definirane četiri glavne tablice: Album, Track, Playlist i pomoćna poveznica Playlist_Track koja omogućava višestruke veze između pjesama i playlista. Svaka klasa predstavlja tablicu u bazi sa svojim stupcima i vezama između njih. Skripta se spaja na MySQL bazu spotify_songs, briše postojeće tablice i kreira nove prema definicijama iz klasa. Zatim iz CSV datoteke izdvoji jedinstvene albume, pjesme i playliste koje se potom ubacuju u bazu. Pjesme su povezane s albumima i playlistama kroz definirane odnose u modelu. Na kraju se ispisuje poruka o uspješnom popunjavanju baze, a cijeli proces omogućuje strukturirano i relacijsko spremanje velikog skupa glazbenih podataka za kasniju analizu i upite.

