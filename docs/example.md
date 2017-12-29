## Examples

This is a very basic example on how to implement a nearly-incompressible version of the Neo-Hookean material model in a commercial FEM package (HYPELA2 for MSC.Marc).

The formulation of the second Piola-Kirchhoff stress:

<a href="https://www.codecogs.com/eqnedit.php?latex=\mathbf{S}&space;=&space;2\text{C}_{10}&space;\&space;\text{dev}(\hat{\mathbf{C}})&space;\mathbf{C}^{-1}&space;&plus;&space;\kappa&space;(J-1)&space;J&space;\mathbf{C}^{-1}" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\mathbf{S}&space;=&space;2\text{C}_{10}&space;\&space;\text{dev}(\hat{\mathbf{C}})&space;\mathbf{C}^{-1}&space;&plus;&space;\kappa&space;(J-1)&space;J&space;\mathbf{C}^{-1}" title="\mathbf{S} = 2\text{C}_{10} \ \text{dev}(\hat{\mathbf{C}}) \mathbf{C}^{-1} + \kappa (J-1) J \mathbf{C}^{-1}" /></a>

By evaluating the derivative of the stress with respect to the right Cauchy-Green deformation tensor we get the material elasticity tensor:

<a href="https://www.codecogs.com/eqnedit.php?latex=\mathbb{C}&space;=&space;2\text{C}_{10}&space;J^{-2/3}&space;\frac{2}{3}&space;\&space;(\text{tr}(\mathbf{C})&space;\&space;\mathbb{I}&space;-&space;\mathbf{1}&space;\otimes&space;\mathbf{C}^{-1}&space;-&space;\mathbf{C}^{-1}&space;\otimes&space;\mathbf{1}&space;&plus;&space;\frac{1}{3}&space;\text{tr}(\mathbf{C})&space;\&space;\mathbf{C}^{-1}&space;\otimes&space;\mathbf{C}^{-1})&space;&plus;&space;\left(\kappa&space;(J-1)&space;J&space;&plus;&space;\kappa&space;J^2\right)&space;\&space;\mathbf{C}^{-1}&space;\otimes&space;\mathbf{C}^{-1}&space;-&space;2&space;\kappa&space;(J-1)&space;J&space;\&space;\mathbb{I}" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\mathbb{C}&space;=&space;2\text{C}_{10}&space;J^{-2/3}&space;\frac{2}{3}&space;\&space;(\text{tr}(\mathbf{C})&space;\&space;\mathbb{I}&space;-&space;\mathbf{1}&space;\otimes&space;\mathbf{C}^{-1}&space;-&space;\mathbf{C}^{-1}&space;\otimes&space;\mathbf{1}&space;&plus;&space;\frac{1}{3}&space;\text{tr}(\mathbf{C})&space;\&space;\mathbf{C}^{-1}&space;\otimes&space;\mathbf{C}^{-1})&space;&plus;&space;\left(\kappa&space;(J-1)&space;J&space;&plus;&space;\kappa&space;J^2\right)&space;\&space;\mathbf{C}^{-1}&space;\otimes&space;\mathbf{C}^{-1}&space;-&space;2&space;\kappa&space;(J-1)&space;J&space;\&space;\mathbb{I}" title="\mathbb{C} = 2\text{C}_{10} J^{-2/3} \frac{2}{3} \ (\text{tr}(\mathbf{C}) \ \mathbb{I} - \mathbf{1} \otimes \mathbf{C}^{-1} - \mathbf{C}^{-1} \otimes \mathbf{1} + \frac{1}{3} \text{tr}(\mathbf{C}) \ \mathbf{C}^{-1} \otimes \mathbf{C}^{-1}) + \left(\kappa (J-1) J + \kappa J^2\right) \ \mathbf{C}^{-1} \otimes \mathbf{C}^{-1} - 2 \kappa (J-1) J \ \mathbb{I}" /></a>

The two equations are now implemented in a Total Lagrange user subroutine as follows:

```fortran
      include 'ttb/ttb_library.f'

      subroutine hypela2(d,g,e,de,s,t,dt,ngens,m,nn,kcus,matus,ndi,
     2             nshear,disp,dispt,coord,ffn,frotn,strechn,eigvn,ffn1,
     3                   frotn1,strechn1,eigvn1,ncrd,itel,ndeg,ndm,
     4                   nnode,jtype,lclass,ifr,ifu)
     
      ! HYPELA2 Nearly-Incompressible Neo-Hookean Material (3D analysis)
      ! Formulation: Total Lagrange
      ! Example for usage of Tensor Toolbox
      !
      ! Switch to Voigt Notation:
      ! - change commented Tensor Datatypes
      !
      ! Andreas Dutzler
      ! 2017-12-27
      ! Graz University of Technology

      use Tensor
      implicit none
      
      real*8 coord, d, de, disp, dispt, dt, e, eigvn, eigvn1, ffn, ffn1
      real*8 frotn, frotn1, g
      integer ifr, ifu, itel, jtype, kcus, lclass, matus, m, ncrd, ndeg
      integer ndi, ndm, ngens, nn, nnode, nshear
      real*8 s, strechn, strechn1, t

      dimension e(*),de(*),t(*),dt(*),g(*),d(ngens,*),s(*)
      dimension m(2),coord(ncrd,*),disp(ndeg,*),matus(2),
     *          dispt(ndeg,*),ffn(itel,3),frotn(itel,3),
     *          strechn(itel),eigvn(itel,*),ffn1(itel,3),
     *          frotn1(itel,3),strechn1(itel),eigvn1(itel,*),
     *          kcus(2),lclass(2)

      type(Tensor2)  :: F1
      real(kind=8) :: J,kappa,C10
      
      ! voigt notation: change to type Tensor2s, Tensor4s
      type(Tensor2) :: C1,S1,Eye
      type(Tensor4) :: C4
      
      ! material parameters
      C10 = 0.5
      kappa = 500.0
      
      Eye = identity2(Eye)
      F1 = ffn1(1:3,1:3)
      J = det(F1)
      
      ! right cauchy-green tensor
      C1 = transpose(F1)*F1
      
      ! pk2 stress
      S1 = 2.*C10*J**(-2./3.)*dev(C1)*inv(C1) + kappa*(J-1)*J*inv(C1)

      ! material elasticity tensor
      C4 = 2.*C10 * J**(-2./3.) * 2./3. *
     *     (tr(C1) * (inv(C1).cdya.inv(C1))
     *    -(Eye.dya.inv(C1))-(inv(C1).dya.Eye)
     *    +tr(C1)/3.*(inv(C1).dya.inv(C1)))
     *    +(kappa*(J-1)*J+kappa*J**2)*(inv(C1).dya.inv(C1))
     *    -2.*kappa*(J-1)*J*(inv(C1).cdya.inv(C1))
     
      ! output as array
      s(1:ngens)         = asarray( voigt(S1), ngens )
      d(1:ngens,1:ngens) = asarray( voigt(C4), ngens, ngens )
      
      return
      end
```

There are also examples for a [basic understandig of the tensor toolbox](examples/script_umat.f) and a [full featured MSC.Marc Neo-Hookean material HYPELA2 user subroutine](examples/hypela2_nh_ttb.f).