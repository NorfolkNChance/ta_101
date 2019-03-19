UL Maya Python tests

**TBD** Grade these for difficulty




- Write a function that returns all of the skinned meshes in the scene.  Make sure that the results don't include intermediate meshes.

	a. good solution starts with skinclusters and works forward, rather than starting with meshes
	b. good solution filters intermediate results properly
	c. good solution is exception safe for no skins, only intermediate skins

	bonus points:  return long paths



- Write a function which finds all of the meshes in the world whose grandparents are children of the world.  It's possible that some of the meshes will be instanced (ie, they will be parented to more than one transform at the same time).  Return the full path of the meshes.

	a. a good solution will use  `ls(type=mesh, ap=True)` to find instanced meshes 
	b. an very good solution will count pipes to find the hierarchy depth
	c. an excellent solution will use set() to deduplicated the results


	Minimal solution:

    def find_grandchild_meshes()
	    meshes =  cmds.ls(type='mesh', ap=True)
	    mesh_parents = cmds.listRelatives(p, ap=True, f=True)
	    grandchildren = set([i for i in mesh_parents if i.count("|") == 2])
        return cmds.listRelatives(*grandchildren, s=True, f=True)


- Given this skeleton write a function which adds and enables IK handles on the arms without causing the pose to change.  
    a. Main task is to find the plane of the IK problably by taking a triangle using elbow, shoulder, and wrist.  There are multiple possible locations; a passable answer is to locate the IK at the elbow but a better one projects it out behind the elbow.
    b. deliberately make the skeleton have mismatched names on left and right, but have correct labels. See if candidate catches that.
    c. **advanced, maybe optional** does the function know how to deal with a hyperextended arm?



- Write a function when given a reference point, creates a nurbs circle which is oriented normal to the point and whose radius is encompasses the all of the mesh vertices: essentially, you want the  minimal circle which contains all of the mesh points from the perspective of the reference point.  

	a. solving this involves four steps
		A. get the centroid of the mesh
		B. create a transform whose origin is the centroid and whose up axis is along the line from the reference to the centroid
		C. get the position of all of the mesh points in that space, with their up-axis dimension zeroed out
		D. The largest vertex distance is the radius of the circle. It's orientation is the same as the transform from B
	b. this could be solved by purely mathematical and API means, or by using nested transforms and constraints.  Either is acceptable.

- Write a function that will find all the meshes in a scene which are overlapped in world space.  This is a check for visual overlap, it's possible that meshes might have the same vertices in world space but different transform hierarchies.  The check should be accurate to a tolerance of 0.1cm.  There may be a large number of meshes, or meshes with a lot of vertices.  The result should allow you to iterate over all the clusters of overlapped meshes.  This code only needs to find exact vertex-for-vertex overlays, not intersections or 

    a. a good solution will be roughly linear in time.
    b. a better solution will use some kind of fast pre-processing check to skip meshes that could not possibly overlap others, using bboxes, octrees, or other quick rejection methods.
    c. an excellent solution will probably use a hashing function to generate a repeatable hash for a given world space mesh
    d. Look for code that minimizes the memory footprint of the operation as it it goes along

- Write a function which finds the angle of the elbow on this skeleton in degrees. Due to the joint orient / rotate orient setup this will not be the euler angle of the visible skeleton
    a. basically, this is a dot product.  
    b. could they hack it with some other trick, like RO offset?
    c. do they use API or other tricks?

-