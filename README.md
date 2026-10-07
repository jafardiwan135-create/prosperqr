<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>ProsperQR Router</title>
  <style>
    * { box-sizing: border-box; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
    body { background-color: #f4f6f8; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }
    .card { background: #ffffff; padding: 24px; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); width: 100%; max-width: 400px; text-align: center; }
    h2 { margin-top: 0; color: #1a1a1a; font-size: 20px; }
    p { color: #666; font-size: 14px; line-height: 1.5; }
    input { width: 100%; padding: 12px; margin: 12px 0; border: 1px solid #ccc; border-radius: 8px; font-size: 14px; box-sizing: border-box; }
    button { width: 100%; padding: 12px; background-color: #0070f3; color: white; border: none; border-radius: 8px; font-weight: bold; font-size: 15px; cursor: pointer; }
    button:disabled { background-color: #a0c4ff; }
    .status-box { padding: 16px; border-radius: 8px; background: #eef6ff; color: #0050b3; font-weight: 500; }
    .success-box { background: #e6f4ea; color: #137333; }
    .error-box { background: #fce8e6; color: #c5221f; }
    .hidden { display: none; }
  </style>

  <!-- Supabase JS Client library -->
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
</head>
<body>

  <div class="card">
    <!-- Screen 1: Loading State -->
    <div id="loading-screen" class="status-box">
      Checking QR destination...
    </div>

    <!-- Screen 2: Setup Screen (If destination is NULL) -->
    <div id="setup-screen" class="hidden">
      <h2>Link Google Business</h2>
      <p>This QR code (<b id="display-qr-id"></b>) is unassigned. Enter the business <b>Google Place ID</b> below to bind it.</p>
      
      <input type="text" id="place-id-input" placeholder="e.g. ChIJN1t_tDeuEmsRUsoyG83frY4" />
      <button id="save-btn" onclick="saveBusinessLink()">Save & Link QR</button>
    </div>

    <!-- Screen 3: Success Message -->
    <div id="success-screen" class="status-box success-box hidden">
      ✅ Business linked successfully! Redirection active.
    </div>

    <!-- Screen 4: Error Screen -->
    <div id="error-screen" class="status-box error-box hidden">
      ❌ Invalid QR Code parameter.
    </div>
  </div>

  <script>
    // 1. YOUR SUPABASE CREDENTIALS (PASTE YOURS HERE)
    const SUPABASE_URL = "https://jshbvklcvzpzmkrvnaoj.supabase.co"; 
    const SUPABASE_ANON_KEY = "sb_publishable_6i2KNjaLhcasDx4lu-xa0w_dh-iqyxQ";

    // Initialize Supabase Client
    const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

    // Get QR ID from URL query parameters (e.g. ?id=qr_001)
    const urlParams = new URLSearchParams(window.location.search);
    const qrId = urlParams.get('id');

    async function handleQRScan() {
      if (!qrId) {
        showScreen('error-screen');
        document.getElementById('error-screen').innerText = "❌ No QR ID found in URL. Use format: ?id=YOUR_ID";
        return;
      }

      // Query database for this QR code
      const { data, error } = await supabase
        .from('qr_codes')
        .select('*')
        .eq('id', qrId)
        .single();

      if (error || !data) {
        showScreen('error-screen');
        document.getElementById('error-screen').innerText = `❌ QR Code '${qrId}' does not exist in database.`;
        return;
      }

      // Logic Branching based on flowchart
      if (data.destination_url) {
        // If available -> Open destination link immediately
        window.location.href = data.destination_url;
      } else {
        // If not available -> Open add business screen
        document.getElementById('display-qr-id').innerText = qrId;
        showScreen('setup-screen');
      }
    }

    async function saveBusinessLink() {
      const placeId = document.getElementById('place-id-input').value.trim();
      const saveBtn = document.getElementById('save-btn');

      if (!placeId) {
        alert('Please enter a valid Place ID');
        return;
      }

      saveBtn.innerText = "Saving...";
      saveBtn.disabled = true;

      // Construct Google Review Link (Base Link + Place ID)
      const generatedReviewUrl = `https://search.google.com/local/writereview?placeid=${placeId}`;

      // Update Database
      const { error } = await supabase
        .from('qr_codes')
        .update({ destination_url: generatedReviewUrl, place_id: placeId })
        .eq('id', qrId);

      if (error) {
        alert('Error saving destination link: ' + error.message);
        saveBtn.innerText = "Save & Link QR";
        saveBtn.disabled = false;
      } else {
        // Show message "Added"
        showScreen('success-screen');
        
        // Auto-redirect to review page after 2 seconds
        setTimeout(() => {
          window.location.href = generatedReviewUrl;
        }, 2000);
      }
    }

    function showScreen(screenId) {
      document.getElementById('loading-screen').classList.add('hidden');
      document.getElementById('setup-screen').classList.add('hidden');
      document.getElementById('success-screen').classList.add('hidden');
      document.getElementById('error-screen').classList.add('hidden');
      document.getElementById(screenId).classList.remove('hidden');
    }

    // Run scanner logic on load
    handleQRScan();
  </script>
</body>
</html>
