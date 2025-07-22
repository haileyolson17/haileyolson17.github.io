# haileyolson17.github.io
import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { useState } from "react";
import { loadStripe } from "@stripe/stripe-js";

const stripePromise = loadStripe("pk_test_XXXXXXXXXXXXXXXXXXXXXXXX"); // Replace with your real Stripe publishable key

export default function SmartMatchLanding() {
  const [location, setLocation] = useState("");
  const [budget, setBudget] = useState("");
  const [bedrooms, setBedrooms] = useState("");
  const [bathrooms, setBathrooms] = useState("");
  const [squareFeet, setSquareFeet] = useState("");
  const [submitted, setSubmitted] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();

    const stripe = await stripePromise;

    const response = await fetch("/api/create-checkout-session", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        location,
        budget,
        bedrooms,
        bathrooms,
        squareFeet,
      }),
    });

    const session = await response.json();
    await stripe.redirectToCheckout({ sessionId: session.id });
  };

  return (
    <div className="p-8 max-w-3xl mx-auto">
      <h1 className="text-4xl font-bold mb-4">SmartMatch Home Report</h1>
      <p className="text-lg mb-6">
        Discover homes that match your goals and find agents who’ve delivered results. Get your custom report for just <span className="font-semibold">$47</span>.
      </p>

      <Card className="mb-6">
        <CardContent className="p-6">
          {submitted ? (
            <div>
              <h2 className="text-2xl font-semibold mb-2">Thank you!</h2>
              <p>We'll start working on your custom report and deliver it within 48 hours.</p>
            </div>
          ) : (
            <form onSubmit={handleSubmit} className="space-y-4">
              <Input
                placeholder="Preferred Location (City or Zip)"
                value={location}
                onChange={(e) => setLocation(e.target.value)}
                required
              />
              <Input
                placeholder="Target Budget (e.g. 450000)"
                value={budget}
                onChange={(e) => setBudget(e.target.value)}
                required
              />
              <Input
                placeholder="Minimum Bedrooms (e.g. 3)"
                value={bedrooms}
                onChange={(e) => setBedrooms(e.target.value)}
                required
              />
              <Input
                placeholder="Minimum Bathrooms (e.g. 2)"
                value={bathrooms}
                onChange={(e) => setBathrooms(e.target.value)}
              />
              <Input
                placeholder="Minimum Square Feet (e.g. 1500)"
                value={squareFeet}
                onChange={(e) => setSquareFeet(e.target.value)}
              />
              <Button type="submit" className="w-full">
                Generate My Report for $47
              </Button>
            </form>
          )}
        </CardContent>
      </Card>

      <div className="text-sm text-muted-foreground">
        Need help? Contact our support team any time.
      </div>
    </div>
  );
}
<distributionManagement>
   <repository>
     <id>github</id>
     <name>GitHub Hailey Olson Apache Maven Packages</name>
     <url>https://maven.pkg.github.com/OWNER/smarthomematch</url>
   </repository>
</distributionManagement>
// pages/api/create-checkout-session.js

import Stripe from 'stripe';
import { writeFileSync, mkdirSync, existsSync } from 'fs';
import path from 'path';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end('Method Not Allowed');

  const { location, budget, bedrooms, bathrooms, squareFeet } = req.body;

  // Save user data to a simple JSON file (or database)
  const submissionsDir = path.join(process.cwd(), 'submissions');
  if (!existsSync(submissionsDir)) mkdirSync(submissionsDir);

  const filename = `${Date.now()}-${location.replace(/\\s+/g, '_')}.json`;
  const filepath = path.join(submissionsDir, filename);
  writeFileSync(filepath, JSON.stringify(req.body, null, 2));

  // Create a Stripe Checkout Session
  const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    line_items: [{
      price_data: {
        currency: 'usd',
        product_data: {
          name: 'SmartMatch Home Report',
        },
        unit_amount: 4700, // $47.00
      },
      quantity: 1,
    }],
    mode: 'payment',
    success_url: `${req.headers.origin}/thank-you`,
    cancel_url: `${req.headers.origin}/`,
    metadata: {
      submission_filename: filename,
    },
  });

  res.status(200).json({ id: session.id });
}
// pages/thank-you.js
export default function ThankYou() {
  return (
    <div className="p-8 max-w-2xl mx-auto text-center">
      <h1 className="text-3xl font-bold mb-4">Thank you for your purchase!</h1>
      <p className="text-lg">
        We’ve received your request. Your custom SmartMatch Home Report will be delivered within 48 hours.
      </p>
    </div>
  );
}
// pages/api/stripe-webhook.js
import { buffer } from 'micro';
import Stripe from 'stripe';
import fs from 'fs';
import path from 'path';
import generateReport from '../../utils/generateReport'; // You’ll build this

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

export const config = {
  api: {
    bodyParser: false,
  },
};

export default async function handler(req, res) {
  const sig = req.headers['stripe-signature'];
  const rawBody = await buffer(req);
  let event;

  try {
    event = stripe.webhooks.constructEvent(rawBody, sig, process.env.STRIPE_WEBHOOK_SECRET);
  } catch (err) {
    return res.status(400).send(`Webhook error: ${err.message}`);
  }

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object;
    const filename = session.metadata.submission_filename;

    const filePath = path.join(process.cwd(), 'submissions', filename);
    const formData = JSON.parse(fs.readFileSync(filePath, 'utf-8'));

    await generateReport(formData); // Generate and email the report
  }

  res.status(200).json({ received: true });
}
// utils/generateReport.js
import PDFDocument from 'pdfkit';
import fs from 'fs';
import path from 'path';
import nodemailer from 'nodemailer';

export default async function generateReport(formData) {
  const { location, budget, bedrooms, bathrooms, squareFeet } = formData;

  // 1. Generate PDF
  const doc = new PDFDocument();
  const filePath = path.join(process.cwd(), 'reports', `${Date.now()}-${location}.pdf`);
  doc.pipe(fs.createWriteStream(filePath));

  doc.fontSize(18).text('SmartMatch Home Report', { align: 'center' });
  doc.moveDown();

  doc.fontSize(12).text(`Location: ${location}`);
  doc.text(`Budget: $${budget}`);
  doc.text(`Bedrooms: ${bedrooms}`);
  doc.text(`Bathrooms: ${bathrooms}`);
  doc.text(`Square Feet: ${squareFeet}`);
  doc.moveDown();

  doc.fontSize(14).text('Recent Home Sales (simulated data):');
  for (let i = 1; i <= 3; i++) {
    doc.text(`- House ${i}: $${parseInt(budget) - i * 10000}, ${bedrooms}BR / ${bathrooms || 2}BA`);
  }

  doc.end();

  // 2. Send Email with Report (replace with your SendGrid/Nodemailer config)
  const transporter = nodemailer.createTransport({
    service: 'gmail',
    auth: {
      user: process.env.EMAIL_USER,
      pass: process.env.EMAIL_PASS,
    },
  });

  await transporter.sendMail({
    from: `"SmartMatch Reports" <${process.env.EMAIL_USER}>`,
    to: formData.email || 'you@example.com', // Add `email` to form later
    subject: 'Your SmartMatch Home Report',
    text: 'Attached is your custom report.',
    attachments: [
      {
        filename: 'SmartMatchReport.pdf',
        path: filePath,
      },
    ],
  });
}
const [email, setEmail] = useState("");

// inside the form:
<Input
  placeholder="Your Email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  required
/>
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
EMAIL_USER=youremail@example.com
EMAIL_PASS=your_email_password_or_app_password
