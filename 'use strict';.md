'use strict';

// SQLite einbinden
const sqlite3 = require('sqlite3').verbose();

// Express und weitere Module einbinden
const express = require('express');
const bodyParser = require('body-parser');
const cookieParser = require('cookie-parser');
const validator = require("email-validator");

// Verbindung zur SQLite-Datenbank herstellen
const db = new sqlite3.Database('./borkenbank.db', (err) => {
    if (err) {
        console.error('Fehler beim Verbinden mit der Datenbank:', err.message);
    } else {
        console.log('Verbindung zur SQLite-Datenbank hergestellt.');
    }
});

// Tabelle für Kunden anlegen, falls sie noch nicht existiert
db.serialize(() => {
    db.run(`
        CREATE TABLE IF NOT EXISTS kunden (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            nachname TEXT,
            vorname TEXT,
            benutzername TEXT UNIQUE,
            kennwort TEXT,
            istEingeloggt BOOLEAN
        )
    `, (err) => {
        if (err) {
            console.error('Fehler beim Erstellen der Tabelle:', err.message);
        } else {
            console.log('Tabelle "kunden" bereit.');
            // Beispielkunde einfügen, falls nicht vorhanden
            db.get(
                `SELECT * FROM kunden WHERE benutzername = ?`,
                ['pk'],
                (err, row) => {
                    if (err) {
                        console.error('Fehler beim Prüfen des Beispielkunden:', err.message);
                    } else if (!row) {
                        db.run(
                            `INSERT INTO kunden (nachname, vorname, benutzername, kennwort, istEingeloggt)
                             VALUES (?, ?, ?, ?, ?)`,
                            ['Kiff', 'Pit', 'pk', '123', false],
                            (err) => {
                                if (err) {
                                    console.error('Fehler beim Einfügen des Beispielkunden:', err.message);
                                } else {
                                    console.log('Beispielkunde wurde angelegt.');
                                }
                            }
                        );
                    } else {
                        console.log('Beispielkunde existiert bereits.');
                    }
                }
            );
        }
    });
});

// Klassendefinition des Kunden
class Kunde {
    constructor() {
        this.Nachname = "";
        this.Vorname = "";
        this.Benutzername = "";
        this.Kennwort = "";
        this.IstEingeloggt = false;
        this.Mail = "";
    }
}

// Kundenobjekt deklariert und instanziiert
let kunde = new Kunde();
kunde.Nachname = "Kiff";
kunde.Vorname = "Pit";
kunde.Benutzername = "pk";
kunde.Kennwort = "123";
kunde.IstEingeloggt = false;

// Klassenefinition des Kundenberaters
class Kundenberater {
    constructor() {
        this.Nachname = "";
        this.Vorname = "";
        this.Telefonnummer = "";
        this.Mail = "";
        this.Bild = "";
    }
}

// Deklaration und Instanziierung
let kundenberater = new Kundenberater();
kundenberater.Nachname = "Pass";
kundenberater.Vorname = "Hildegard";
kundenberater.Telefonnummer = "012345 67890";
kundenberater.Mail = "h.pass@borken-bank.de";
kundenberater.Bild = "pass.jpg";

// Server-Konfiguration
const PORT = 3000;
const HOST = '0.0.0.0';
const app = express();

app.use(express.static('public'));
app.set('view engine', 'ejs');
app.use(bodyParser.urlencoded({ extended: true }));
app.use(cookieParser());

// Geheimer Schlüssel für signierte Cookies (optional, aktuell nicht genutzt)
const secretKey = 'mein_geheimer_schluessel';
// app.use(cookieParser(secretKey));

// Routing

app.get('/', (req, res) => {
    if (kunde.IstEingeloggt) {
        res.render('index.ejs', {});
    } else {
        res.render('login.ejs', {
            Meldung: "Melden Sie sich zuerst an."
        });
    }
});

app.get('/agb', (req, res) => {
    if (kunde.IstEingeloggt) {
        res.render('agb.ejs', {});
    } else {
        res.render('login.ejs', {
            Meldung: "Melden Sie sich zuerst an."
        });
    }
});

app.get('/hilfe', (req, res) => {
    if (kunde.IstEingeloggt) {
        res.render('hilfe.ejs', {});
    } else {
        res.render('login.ejs', {
            Meldung: "Melden Sie sich zuerst an."
        });
    }
});

app.get('/kontenuebersicht', (req, res) => {
    if (kunde.IstEingeloggt) {
        res.render('kontenuebersicht.ejs', {});
    } else {
        res.render('login.ejs', {
            Meldung: "Melden Sie sich zuerst an."
        });
    }
});

app.get('/profil', (req, res) => {
    if (kunde.IstEingeloggt) {
        res.render('profil.ejs', {
            Meldung: "",
            Email: kunde.Mail
        });
    } else {
        res.render('login.ejs', {
            Meldung: "Melden Sie sich zuerst an."
        });
    }
});

app.post('/profil', (req, res) => {
    let meldung = "";
    if (kunde.IstEingeloggt) {
        let email = req.body.Email;
        if (validator.validate(email)) {
            console.log("Gültige EMail.");
            meldung = "EMail-adresse gültig";
            kunde.Mail = email;
        } else {
            console.log("Ungültige EMail.");
            meldung = "EMail-adresse ungültig";
        }
        res.render('profil.ejs', {
            Meldung: meldung,
            Email: kunde.Mail
        });
    } else {
        res.render('login.ejs', {
            Meldung: "Melden Sie sich zuerst an."
        });
    }
});

app.get('/postfach', (req, res) => {
    res.render('postfach.ejs', {});
});

app.get('/kreditBeantragen', (req, res) => {
    if (kunde.IstEingeloggt) {
        res.render('kreditBeantragen.ejs', {
            Laufzeit: "",
            Zinssatz: "",
            Betrag: "",
            Meldung: ""
        });
    } else {
        res.render('login.ejs', {
            Meldung: "Melden Sie sich zuerst an."
        });
    }
});

app.post('/kreditBeantragen', (req, res) => {
    let zinsbetrag = req.body.Betrag;
    let laufzeit = req.body.Laufzeit;
    let zinssatz = req.body.Zinssatz;
    let kredit = zinsbetrag * Math.pow(1 + zinssatz / 100, laufzeit);
    console.log("Rückzahlungsbetrag: " + kredit + " €.");
    res.render('kreditBeantragen.ejs', {
        Laufzeit: laufzeit,
        Zinssatz: zinssatz,
        Betrag: zinsbetrag,
        Meldung: "Rückzahlungsbetrag: " + kredit + " €."
    });
});

app.get('/ueberweisungAusfuehren', (req, res) => {
    if (kunde.IstEingeloggt) {
        res.render('ueberweisungAusfuehren.ejs', {});
    } else {
        res.render('login.ejs', {
            Meldung: "Melden Sie sich zuerst an."
        });
    }
});

app.get('/geldAnlegen', (req, res) => {
    if (kunde.IstEingeloggt) {
        res.render('geldAnlegen.ejs', {
            Betrag: 120,
            Laufzeit: 2,
            Meldung: ""
        });
    } else {
        res.render('login.ejs', {
            Meldung: "Melden Sie sich zuerst an."
        });
    }
});

app.post('/geldAnlegen', (req, res) => {
    let betrag = req.body.Betrag;
    let laufzeit = req.body.Laufzeit;
    let zinssatz = 0.1;
    let zinsen = betrag * zinssatz;
    if (kunde.IstEingeloggt) {
        res.render('geldAnlegen.ejs', {
            Betrag: betrag,
            Laufzeit: laufzeit,
            Meldung: "Ihre Zinsen betragen: " + zinsen
        });
    } else {
        res.render('login.ejs', {
            Meldung: "Melden Sie sich zuerst an."
        });
    }
});

app.get('/login', (req, res) => {
    kunde.IstEingeloggt = false;
    res.render('login.ejs', {
        Meldung: "Bitte Benutzername und Kennwort eingeben."
    });
});

app.post('/login', (req, res) => {
    let benutzername = req.body.IdKunde;
    let kennwort = req.body.Kennwort;
    let meldung = "";
    if (kunde.Benutzername == benutzername && kunde.Kennwort == kennwort) {
        meldung = "Die Zugangsdaten wurden korrekt eingegeben";
        kunde.IstEingeloggt = true;
        res.cookie('istAngemeldetAls', JSON.stringify(kunde), { maxAge: 900000, httpOnly: true, signed: false });
        res.render('index.ejs', {
            Meldung: meldung
        });
    } else {
        meldung = "Die Zugangsdaten wurden NICHT korrekt eingegeben.";
        kunde.IstEingeloggt = false;
        res.render('login.ejs', {
            Meldung: meldung
        });
    }
});

// Server starten
app.listen(PORT, HOST, () => {
    console.log(`Running on http://${HOST}:${PORT}`);
});