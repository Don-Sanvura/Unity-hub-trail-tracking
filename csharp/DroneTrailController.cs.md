[[Drone trail]]

using System;
using System.Collections;
using UnityEngine;

/// <summary>
/// Attach to the drone. Provides:
///  - live trail via TrailRenderer (set up in inspector)
///  - method SnapshotAndClear() to convert the current TrailRenderer into a fading LineRenderer (world-space),
///    then clears the TrailRenderer so future movement doesn't connect to old trail.
/// </summary>
[RequireComponent(typeof(Transform))]
public class DroneTrailController : MonoBehaviour
{
    [Header("References")]
    [Tooltip("Reference to the TrailRenderer component on the drone (live trail).")]
    public TrailRenderer trailRenderer;

    [Tooltip("Prefab that contains a LineRenderer (useWorldSpace = true) and material that supports alpha.")]
    public GameObject linePrefab; // prefab must have a LineRenderer component

    [Header("Fade settings")]
    [Tooltip("How long the stamped trail will take to fade (seconds).")]
    public float fadeDuration = 4f;

    [Tooltip("Minimum number of points required to create a stamped line.")]
    public int minPointsToStamp = 2;

    // Optional: automatically stamp when teleporting or stopping (you can call SnapshotAndClear manually)
    void Reset()
    {
        fadeDuration = 4f;
        minPointsToStamp = 2;
    }

    /// <summary>
    /// Converts the current TrailRenderer positions into a LineRenderer object that fades and destroys itself.
    /// Then clears the TrailRenderer so it doesn't connect to new positions (useful for teleporting).
    /// Call this before you teleport the drone.
    /// </summary>
    public void SnapshotAndClear()
    {
        if (trailRenderer == null || linePrefab == null)
        {
            Debug.LogWarning("DroneTrailController: trailRenderer or linePrefab is not assigned.");
            return;
        }

        // Get positions from TrailRenderer into a buffer
        // Use a large buffer and rely on returned count to avoid version-specific properties
        Vector3[] buffer = new Vector3[2048];
        int count = trailRenderer.GetPositions(buffer); // returns number of positions written
        if (count < minPointsToStamp)
        {
            // Nothing meaningful to stamp
            trailRenderer.Clear();
            return;
        }

        Vector3[] positions = new Vector3[count];
        Array.Copy(buffer, positions, count);

        // Create a new line GameObject from prefab and set positions
        GameObject go = Instantiate(linePrefab, Vector3.zero, Quaternion.identity);
        LineRenderer lr = go.GetComponent<LineRenderer>();
        if (lr == null)
        {
            Debug.LogError("DroneTrailController: linePrefab must have a LineRenderer component.");
            Destroy(go);
            return;
        }

        lr.positionCount = positions.Length;
        lr.SetPositions(positions);

        // Ensure the material instance is unique so fading won't affect shared material
        Material matInstance = null;
        if (lr.material != null)
        {
            matInstance = new Material(lr.material);
            lr.material = matInstance;
        }
        else if (lr.sharedMaterial != null)
        {
            matInstance = new Material(lr.sharedMaterial);
            lr.material = matInstance;
        }

        // Start fading coroutine
        if (matInstance != null)
        {
            StartCoroutine(FadeAndDestroyCoroutine(matInstance, go, fadeDuration));
        }
        else
        {
            // fallback: if no material, simply destroy after fade duration
            Destroy(go, fadeDuration);
        }

        // Clear the live trail so it doesn't connect to future positions (teleport safety)
        trailRenderer.Clear();
    }

    /// <summary>
    /// Gradually reduces material alpha and destroys the GameObject when done.
    /// </summary>
    private IEnumerator FadeAndDestroyCoroutine(Material mat, GameObject go, float duration)
    {
        // Try to get a color property. We'll assume the material uses "_Color".
        // If it uses a different property name adjust accordingly.
        Color initialColor = Color.white;
        string colorProp = "_Color";
        if (mat.HasProperty(colorProp))
        {
            initialColor = mat.GetColor(colorProp);
        }
        else
        {
            // try mainColor fallback
            try
            {
                initialColor = mat.color;
            }
            catch { }
        }

        float elapsed = 0f;
        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            float t = Mathf.Clamp01(elapsed / duration);
            float alpha = Mathf.Lerp(initialColor.a, 0f, t);
            Color c = initialColor;
            c.a = alpha;

            if (mat.HasProperty(colorProp))
                mat.SetColor(colorProp, c);
            else
                mat.color = c;

            yield return null;
        }

        Destroy(go);
    }

    // Example helper: call this when you teleport the drone
    public void TeleportTo(Vector3 newPosition)
    {
        // Stamp the current trail so it persists (and will fade on its own)
        SnapshotAndClear();

        // Move the drone
        transform.position = newPosition;

        // Optionally, you can also clear trailRenderer again to ensure no artifacts
        trailRenderer.Clear();
    }
}

### Example Teleport / Movement integration

Wherever you teleport the drone in your code, replace your teleport call with:

// Assume drone has a DroneTrailController reference named 'trailCtrl'
trailCtrl.SnapshotAndClear();
transform.position = newPos; // or trailCtrl.TeleportTo(newPos);

This saves the old trail as a fading line and prevents the new position from connecting to the old trail.
