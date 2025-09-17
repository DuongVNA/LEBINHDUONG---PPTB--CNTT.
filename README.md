"type": "TICKET_ERROR",
  "error_code": "TICK-RT-08",
  "reported_at": "2025-09-17T14:00:00+07:00",
  "booking": {
    "roundtrip": true,
    "passengers": 8,
    "origin_city": "Ho Chi Minh City",
    "origin_iata": "SGN",
    "destination_city": "Ha Noi",
    "destination_iata": "HAN",
    "outbound": {
      "flight_no": "VN214",
      "departure_local": "2025-09-17T14:00:00+07:00"
    },
    "return": {
      "date_local": "2025-09-21"
    }
  },
  "financial": {
    "currency": "VND",
    "total_vnd": 64080000,
    "charged_account": {
      "bank": "MB Bank",
      "account_name": "Cty DV DL QC Binh Duong",
      "account_no": "0879479331",
      "masked": "0879•••9331"
    }
  },
  "cause": "Ke toan duyet sai cong thong bao tru tien",
  "status": "OPEN",
  "source": "lebinhduong.vn/ticketing"
}

-- PostgreSQL schema
CREATE TABLE IF NOT EXISTS ticket_error_log (
  id BIGSERIAL PRIMARY KEY,
  logged_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  origin_city      TEXT NOT NULL,
  origin_iata      TEXT NOT NULL,
  destination_city TEXT NOT NULL,
  destination_iata TEXT NOT NULL,
  flight_no_out    TEXT,
  flight_no_ret    TEXT,
  depart_at_local  TIMESTAMPTZ,
  return_date_local DATE,
  passengers       SMALLINT CHECK (passengers > 0),
  is_roundtrip     BOOLEAN NOT NULL DEFAULT TRUE,
  total_amount_vnd NUMERIC(15,0) CHECK (total_amount_vnd >= 0),
  currency         CHAR(3) NOT NULL DEFAULT 'VND',
  cause            TEXT,
  bank_name        TEXT,
  account_name     TEXT,
  payment_account_no_raw TEXT,
  payment_account_no_norm TEXT,
  payment_account_masked TEXT GENERATED ALWAYS AS (
    CASE
      WHEN payment_account_no_norm ~ '^[0-9]{6,}$'
      THEN substr(payment_account_no_norm,1,4) || '•••' || right(payment_account_no_norm,4)
      ELSE NULL
    END
  ) STORED,
  status           TEXT NOT NULL DEFAULT 'OPEN',
  source           TEXT
);

-- Insert bản ghi sự cố của anh
INSERT INTO ticket_error_log (
  origin_city, origin_iata, destination_city, destination_iata,
  flight_no_out, flight_no_ret, depart_at_local, return_date_local,
  passengers, is_roundtrip, total_amount_vnd, currency, cause,
  bank_name, account_name, payment_account_no_raw, payment_account_no_norm,
  status, source
) VALUES (
  'Ho Chi Minh City', 'SGN', 'Ha Noi', 'HAN',
  'VN214', NULL, '2025-09-17 14:00:00+07', '2025-09-21',
  8, TRUE, 64080000, 'VND', 'Ke toan duyet sai cong thong bao tru tien',
  'MB Bank', 'Cty DV DL QC Binh Duong', 'o879479331', '0879479331',
  'OPEN', 'lebinhduong.vn/ticketing'
)
RETURNING id, payment_account_masked;

// npm i pg
const { Client } = require('pg');

async function logTicketError() {
  const client = new Client({
    connectionString: process.env.PG_URL // ví dụ: postgres://user:pass@host:5432/db
  });
  await client.connect();

  await client.query(/*sql*/`
    CREATE TABLE IF NOT EXISTS ticket_error_log (
      id BIGSERIAL PRIMARY KEY,
      logged_at TIMESTAMPTZ NOT NULL DEFAULT now(),
      origin_city TEXT NOT NULL,
      origin_iata TEXT NOT NULL,
      destination_city TEXT NOT NULL,
      destination_iata TEXT NOT NULL,
      flight_no_out TEXT,
      flight_no_ret TEXT,
      depart_at_local TIMESTAMPTZ,
      return_date_local DATE,
      passengers SMALLINT CHECK (passengers > 0),
      is_roundtrip BOOLEAN NOT NULL DEFAULT TRUE,
      total_amount_vnd NUMERIC(15,0) CHECK (total_amount_vnd >= 0),
      currency CHAR(3) NOT NULL DEFAULT 'VND',
      cause TEXT,
      bank_name TEXT,
      account_name TEXT,
      payment_account_no_raw TEXT,
      payment_account_no_norm TEXT,
      payment_account_masked TEXT GENERATED ALWAYS AS (
        CASE
          WHEN payment_account_no_norm ~ '^[0-9]{6,}$'
          THEN substr(payment_account_no_norm,1,4) || '•••' || right(payment_account_no_norm,4)
          ELSE NULL
        END
      ) STORED,
      status TEXT NOT NULL DEFAULT 'OPEN',
      source TEXT
    );
  `);

  const sql = `
    INSERT INTO ticket_error_log (
      origin_city, origin_iata, destination_city, destination_iata,
      flight_no_out, flight_no_ret, depart_at_local, return_date_local,
      passengers, is_roundtrip, total_amount_vnd, currency, cause,
      bank_name, account_name, payment_account_no_raw, payment_account_no_norm,
      status, source
    ) VALUES (
      $1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13,$14,$15,$16,$17,$18
    ) RETURNING id, payment_account_masked;
  `;

  const params = [
    'Ho Chi Minh City', 'SGN', 'Ha Noi', 'HAN',
    'VN214', null, '2025-09-17 14:00:00+07', '2025-09-21',
    8, true, 64080000, 'VND', 'Ke toan duyet sai cong thong bao tru tien',
    'MB Bank', 'Cty DV DL QC Binh Duong', 'o879479331', '0879479331',
    'OPEN', 'lebinhduong.vn/ticketing'
  ];

  const { rows } = await client.query(sql, params);
  console.log('Logged error id:', rows[0].id, 'Masked:', rows[0].payment_account_masked);
  await client.end();
}

logTicketError().catch(console.error);
