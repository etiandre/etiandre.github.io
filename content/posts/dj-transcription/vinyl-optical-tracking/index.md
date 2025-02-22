+++
title = 'Optical tracking of vinyl records'
date = 2024-09-26T14:48:48+02:00
draft = true
+++

We implemented an optical tracking method to measure the angular velocity of a vinyl record during playback, enabling the extraction of the time-warping ground truth from a filmed DJ performance. The method can be especially useful given the vast amount of videos of DJ performances online. By tracking a reference picture of a vinyl record to each frame of a video of a DJ turntable, the record's rotation speed can be estimated.

$$ 2+2 = 4 \frac{A}{B} $$

## Tracking

The tracking is based on the scale-invariant feature transform (SIFT) algorithm @loweDistinctiveImageFeatures2004. The algorithm extracts keypoints from a reference image of the vinyl's label (@fig:vinyl-ref) and from each frame of a video of the vinyl being played. By matching these keypoints, a homography matrix is computed for each frame. This matrix represents the geometric transformation between the reference and the current frame, capturing the rotation, translation, and scale changes (@fig:vinyl-track). The sequence of homography matrices obtained across the frames is stored for further analysis.

{{< figure src="vinyl-ref.jpg" title="Reference image of the vinyl label." >}}


## Extracting the rotation angles from the homography matrix

Each homography matrix

## Rotation computation

The stored homography matrices are then decomposed to extract the rotation angles of the vinyl record using the singular value decomposition method from #cite(<faugerasMOTIONSTRUCTUREMOTION1988>, form: "prose") (section 7). The rotation on the $z$ axis corresponds to the rotation of the vinyl record. The rotation angle sequence is unwrapped then derived to obtain the rotation speed. Then, with knowledge of the nominal speed of the record (e.g. 33 or 45 rpm), the time-warping function can be calculated. Example results are illustrated in @fig:vinyl-results.

```python
def decompose_homography(M):
    if M is None:
        return 0, 0, 0
    r1 = M[0:3, 0]
    r2 = M[0:3, 1]
    r3 = np.cross(r1, r2)
    R = np.column_stack((r1, r2, r3))

    U, S, Vt = np.linalg.svd(R)
    R = np.dot(U, Vt)

    theta_x = np.arctan2(R[2, 1], R[2, 2])
    theta_y = np.arctan2(-R[2, 0], np.sqrt(R[2, 1] ** 2 + R[2, 2] ** 2))
    theta_z = np.arctan2(R[1, 0], R[0, 0])

    return theta_x, theta_y, theta_z
```

Let $\mathbf{M} = \begin{bmatrix} \mathbf{u_1} & \mathbf{u_2} & \mathbf{u_3} \end{bmatrix}$ be a 3x3 homography matrix.

1. What does $\mathbf{u_1}$, $\mathbf{u_2}$ and $\mathbf{u_3}$ represent ?
2. Let $\mathbf{v} = \mathbf{u_1} \times \mathbf{u_2} $ and $\mathbf{R} = \begin{bmatrix} \mathbf{u_1} & \mathbf{u_2} & \mathbf{v} \end{bmatrix}$. What does $\mathbf{R}$ represent ? Justify your answer.
3. Assume singular value decomposition is performed on $\mathbf{R}$, so that $\mathbf{R} = \mathbf{U} \mathbf{S} \mathbf{V}^T$. Let $\mathbf{P} = \mathbf{U} \mathbf{V}^T$. What can be said of $\mathbf{P}$ ?

Given a homography matrix \( \mathbf{M} \), we can decompose it to retrieve the rotation matrix \( \mathbf{R} \) using the following steps:

1. Assume \( \mathbf{M} \) can be expressed as:
   \[
   \mathbf{M} = \begin{bmatrix}
   \mathbf{r}\_1 & \mathbf{r}\_2 & \mathbf{t} \\
   \end{bmatrix}
   \]
   where \( \mathbf{r}\_1 \) and \( \mathbf{r}\_2 \) are the first two columns of the rotation matrix \( \mathbf{R} \), and \( \mathbf{t} \) is the translation component.

2. Compute the third column \( r_3 \) of the rotation matrix as the cross product:
   \[
   r_3 = r_1 \times r_2
   \]

3. Form the estimated rotation matrix:
   \[
   \mathbf{R} = \begin{bmatrix}
   r_1 & r_2 & r_3 \\
   \end{bmatrix}
   \]

4. Perform Singular Value Decomposition (SVD) on \( \mathbf{R} \):
   \[
   \mathbf{R} = \mathbf{U} \mathbf{S} \mathbf{V}^T
   \]

5. Correct \( \mathbf{R} \) to enforce orthonormality:
   \[
   \mathbf{R} = \mathbf{U} \mathbf{V}^T
   \]

6. Extract the Euler angles \( (\theta_x, \theta_y, \theta_z) \) from the rotation matrix \( \mathbf{R} \):
   \[
   \theta_x = \arctan2(R[2, 1], R[2, 2])
   \]
   \[
   \theta_y = \arctan2(-R[2, 0], \sqrt{R[2, 1]^2 + R[2, 2]^2})
   \]
   \[
   \theta_z = \arctan2(R[1, 0], R[0, 0])
   \]

The angles \( \theta_x \), \( \theta_y \), \( \theta_z \) represent the rotation around the respective axes and allow us to retrieve the 3D rotations embedded within the homography transformation.

{{< figure src="rotato.svg" title="Computed angle and rotation speed of the vinyl. In the experiment, the record was played at 33 rpm for the first 30 seconds, then was \"scratched\" by hand for the remaining of the experiment." >}}

The accuracy of the extracted rotational speed is directly dependent on the temporal resolution of the video (usually 30 or 60 images per second), which limits the granularity of the measurements. Additionally, the method is susceptible to noise introduced by the tracking process, which can affect the precision of the rotation and speed calculations.

Similar object tracking techniques could be used to extract ground truth data from any visible physical control of the DJ equipment.
