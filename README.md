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
     <name>GitHub OWNER Apache Maven Packages</name>
     <url>https://maven.pkg.github.com/OWNER/REPOSITORY</url>
   </repository>
</distributionManagement>
