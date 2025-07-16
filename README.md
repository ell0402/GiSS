package com.gisstek.bossapp;

import android.os.Bundle;
import androidx.fragment.app.FragmentActivity;
import com.google.android.gms.maps.*;
import com.google.android.gms.maps.model.*;
import android.graphics.Color;
import com.google.maps.android.PolyUtil;
import org.json.*;
import java.util.*;
import com.android.volley.*;
import com.android.volley.toolbox.*;

public class MapActivity extends FragmentActivity implements OnMapReadyCallback {
    private GoogleMap mMap;
    private final String API_KEY = "YOUR_GOOGLE_API_KEY";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_map);
        SupportMapFragment mapFragment = 
            (SupportMapFragment) getSupportFragmentManager().findFragmentById(R.id.map);
        mapFragment.getMapAsync(this);
    }

    @Override
    public void onMapReady(GoogleMap googleMap) {
        mMap = googleMap;
        LatLng origin = new LatLng(40.7128, -74.0060);
        LatLng destination = new LatLng(34.0522, -118.2437);
        getDirections(origin, destination);
    }

    private void getDirections(LatLng origin, LatLng dest) {
        String url = "https://maps.googleapis.com/maps/api/directions/json?origin=" +
                origin.latitude + "," + origin.longitude +
                "&destination=" + dest.latitude + "," + dest.longitude +
                "&key=" + API_KEY;
        JsonObjectRequest req = new JsonObjectRequest(Request.Method.GET, url, null,
            response -> {
                try {
                    JSONArray routes = response.getJSONArray("routes");
                    JSONObject overview = routes.getJSONObject(0).getJSONObject("overview_polyline");
                    String points = overview.getString("points");
                    drawPolyline(points);
                } catch (JSONException e) { e.printStackTrace(); }
            }, Throwable::printStackTrace);
        Volley.newRequestQueue(this).add(req);
    }

    private void drawPolyline(String encoded) {
        List<LatLng> path = PolyUtil.decode(encoded);
        mMap.addPolyline(new PolylineOptions().addAll(path).width(10).color(Color.BLUE));
    }
}
package com.gisstek.bossapp;

import android.app.Activity;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.widget.*;
import androidx.annotation.Nullable;
import androidx.appcompat.app.AppCompatActivity;
import com.github.dhaval2404.imagepicker.ImagePicker;
import com.google.firebase.auth.FirebaseAuth;
import com.google.firebase.firestore.*;
import com.google.firebase.storage.*;
import java.util.*;

public class UploadDocumentActivity extends AppCompatActivity {
    private ImageView documentPreview;
    private EditText documentTitle;
    private Uri fileUri;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_upload_document);
        documentPreview = findViewById(R.id.documentPreview);
        documentTitle = findViewById(R.id.documentTitle);
        Button uploadBtn = findViewById(R.id.uploadButton);
        uploadBtn.setOnClickListener(v -> ImagePicker.with(this).cameraOnly().start());
    }

    @Override
    protected void onActivityResult(int reqCode, int resultCode, @Nullable Intent data) {
        super.onActivityResult(reqCode, resultCode, data);
        if (resultCode == Activity.RESULT_OK && data != null) {
            fileUri = data.getData();
            documentPreview.setImageURI(fileUri);
            uploadToFirebase();
        }
    }

    private void uploadToFirebase() {
        String uid = FirebaseAuth.getInstance().getUid();
        String filename = "doc_" + System.currentTimeMillis() + ".jpg";
        StorageReference ref = FirebaseStorage.getInstance().getReference("docs/" + uid + "/" + filename);
        ref.putFile(fileUri).addOnSuccessListener(taskSnapshot ->
            ref.getDownloadUrl().addOnSuccessListener(uri -> {
                saveMetaData(uri.toString(), filename);
            }));
    }

    private void saveMetaData(String downloadUrl, String filename) {
        String uid = FirebaseAuth.getInstance().getUid();
        Map<String, Object> data = new HashMap<>();
        data.put("title", documentTitle.getText().toString());
        data.put("url", downloadUrl);
        data.put("uploadedBy", uid);
        data.put("timestamp", FieldValue.serverTimestamp());
        FirebaseFirestore.getInstance()
            .collection("docs").document(uid)
            .collection("driver_docs").add(data)
            .addOnSuccessListener(docRef ->
                Toast.makeText(this, "Uploaded Successfully", Toast.LENGTH_SHORT).show());
    }
}
