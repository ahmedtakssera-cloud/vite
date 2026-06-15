model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  password  String
  role      Role     @default(EMPLOYEE)
  createdAt DateTime @default(now())
}

model Property {
  id          Int      @id @default(autoincrement())
  ref         String   @unique
  ownerName   String
  phone       String
  description String
  images      String[]
  notes       String?
  price       Float
  commission  Float
  status      PropertyStatus @default(AVAILABLE)
  createdAt   DateTime @default(now())

  viewings    Viewing[]
}

model Client {
  id        Int      @id @default(autoincrement())
  name      String
  phone     String
  status    ClientStatus
  notes     String?
  createdAt DateTime @default(now())

  viewings  Viewing[]
}

model Viewing {
  id           Int      @id @default(autoincrement())
  date         DateTime
  dealStatus   DealStatus @default(PENDING)

  clientId     Int
  propertyId   Int

  client       Client   @relation(fields: [clientId], references: [id])
  property     Property @relation(fields: [propertyId], references: [id])
}

enum Role {
  ADMIN
  MANAGER
  EMPLOYEE
}

enum ClientStatus {
  FOLLOW_UP
  INTERESTED
  NOT_INTERESTED
  DEPOSITED
  QUOTED
}

enum PropertyStatus {
  AVAILABLE
  RESERVED
  SOLD
}

enum DealStatus {
  PENDING
  WON
  LOST
}
import jwt from "jsonwebtoken";
import bcrypt from "bcryptjs";
import prisma from "../prisma.js";

export const login = async (req, res) => {
  const { email, password } = req.body;

  const user = await prisma.user.findUnique({ where: { email } });
  if (!user) return res.status(401).json({ error: "Invalid credentials" });

  const valid = await bcrypt.compare(password, user.password);
  if (!valid) return res.status(401).json({ error: "Invalid credentials" });

  const token = jwt.sign(
    { id: user.id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: "7d" }
  );

  res.json({ token, user });
};
export const createProperty = async (req, res) => {
  const count = await prisma.property.count();

  const ref = `A.T${1000 + count + 1}`;

  const property = await prisma.property.create({
    data: {
      ref,
      ...req.body,
    },
  });

  res.json(property);
};
export const createClient = async (req, res) => {
  const client = await prisma.client.create({
    data: req.body,
  });

  res.json(client);
};
export const createViewing = async (req, res) => {
  const { clientId, propertyId, date } = req.body;

  const viewing = await prisma.viewing.create({
    data: {
      clientId,
      propertyId,
      date: new Date(date),
    },
  });

  res.json(viewing);
};
export const dashboardStats = async (req, res) => {
  const clients = await prisma.client.count();
  const properties = await prisma.property.count();
  const sold = await prisma.property.count({
    where: { status: "SOLD" },
  });

  const deals = await prisma.viewing.count({
    where: { dealStatus: "WON" },
  });

  res.json({
    clients,
    properties,
    sold,
    deals,
  });
};
import { useEffect, useState } from "react";

export default function Dashboard() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch("/api/dashboard")
      .then(r => r.json())
      .then(setData);
  }, []);

  if (!data) return "Loading...";

  return (
    <div className="grid grid-cols-4 gap-4 p-6">
      <Card title="Clients" value={data.clients} />
      <Card title="Properties" value={data.properties} />
      <Card title="Sold" value={data.sold} />
      <Card title="Deals Won" value={data.deals} />
    </div>
  );
}

function Card({ title, value }) {
  return (
    <div className="p-4 shadow rounded-xl bg-white">
      <h3>{title}</h3>
      <p className="text-2xl font-bold">{value}</p>
    </div>
  );
}
